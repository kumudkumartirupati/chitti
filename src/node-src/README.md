# Installation

Include [Chitti](https://github.com/NestAway/chitti) in your package.json as dependency and run `npm install`.

# Usage

## 1. Importing proto constants
Consider a rpc method `hellogrpc` which takes a `helloRequest` as a request and returns `helloResponse` as a response
```proto
package testgrpc;
service TestgrpcService{
    rpc hellogrpc (HelloRequest) returns (HelloResponse) {}
}
message HelloRequest {
    string req_message = 1 ;
}
message HelloResponse {
    string res_message = 1;
}
```
>  NOTE: In this example we are using protobufjs to load the JSON file `test.json` equivalent to the above .proto file

### Getting the grpc objects
Instead of using `grpc.loadObject` to load a protobuf.js object as a grpc object, use `RPCImport` method from `chitti`. <br>
`RPCImport` returns equivalent proto constants of the protobuf messages. All the rpc methods of the services loaded through `RPCImport` are promisified methods.
Usage:
```js
import { RPCImport } from 'chitti';
const { TestgrpcService, HelloRequest, HelloResponse } = RPCImport(require("./test.json")).testgrpc;
``` 

## 2. Implementing service handlers and adding the service to server

### Implementation of the rpc methods

```js
import grpc from 'grpc';
import { RPCServer } from 'chitti'; 

class TestService extends TestgrpcService.Service {
    async hellogrpc(req) {
        return { res_message: 'Hello Grpc !' };
    }
}
//create server
const grpc_server = new RPCServer();
// adding rpc methods to service
grpc_server.addService(TestService);
grpc_server.bind('0.0.0.0:8080', grpc.ServerCredentials.createInsecure());
grpc_server.start();
```
Here,`TestgrpcService` is a grpc object loaded through `RPCImport` 

## 3. Creating stub and calling the rpc method
Clients when calling the rpc method need not give any callback function, instead can await on the promise returned by the rpc method

### Creating a stub:
```js
// load 'TestgrpcService' from RPCImport
TestgrpcService.host = '0.0.0.0';
TestgrpcService.port = 8080;
// or
TestgrpcService.host = '0.0.0.0:8080';

TestgrpcService.credentials = grpc.credentials.createInsecure(); 

// Calling the rpc method
(async () => await TestgrpcService.hellogrpc({ req_messsage: 'requesting hellogrpc' });)();
```
A new stub is created everytime when the host and port values are changed.


## 4. Adding Interceptors to client and server
We can add client and server interceptors while creating the server and client stub by passing  the interceptor's class. Both server and client interceptors can be added globally (added to all services/clients in the server) and also specific to a service/client. <br>
[Chitti](https://github.com/NestAway/chitti) provides an easy interface to add server and client interceptors. <br>
`add_handler_interceptor`: to add server interceptors <br>
`add_call_interceptor`: to add client interceptors <br>
### Default Interceptors
* `custom_error_handler_interceptor` : added to all services 
* `custom_error_call_interceptor`: added to all clients

Refer [error handling](https://github.com/NestAway/chitti/tree/documentation/src/node-src#5-custom-error-implementation) for more details

```js
//server interceptors
Chitti.add_handler_interceptor(TestHandleInterceptor);  // global
TestService.add_handler_interceptor(TestHandleInterceptor); // service specific

//client interceptors
Chitti.add_call_interceptor(TestCallInterceptor); //global
TestgrpcService.add_call_interceptor(TestCallInterceptor); // client specific
```

### Other Interceptors
* `StatsDInterceptor`: This interceptor can be added to send the handlers' response time, failure and success metrics to a statsd `UDP` server listening on `STATSD_PORT` (defaults to 8125) at `STATSD_ADDR` (env variables)  <br>
    Usage:
    ```js
    import { StatsDInterceptor } from 'chitti';

    TestService.add_handler_interceptor(StatsDInterceptor);
    ```
    The statsD measurement `chitti_testgrpc_TestgrpcService_response_time` will give the response time for each rpc method (statsd tag), `chitti_testgrpc_TestgrpcService_success` gives the success count of each rpc method and `chitti_testgrpc_TestgrpcService_failure` gives the failure count.

### Implementation of server Interceptor
Any interceptor should extends `HandlerInterceptor`

```js
import { HandlerInterceptor } from 'chitti';
class TestHandleInterceptor extends HandlerInterceptor {
    async call(request, next) {
        try {
            const response = await next(request);
            //write  your own logic
            return response;
        }
        catch (err) {
            throw err;
        }
    }
}

Chitti.add_handler_interceptor(TestHandleInterceptor);  // adding globally
TestService.add_handler_interceptor(TestHandleInterceptor); // adding to specific service
```

### Implementation of client Interceptor
```js
import grpc from 'grpc';
function TestCallInterceptor(options, nextCall) {
    let savedMessage;
    let savedMessageNext;
    const requester = {
        start(metadata, listener, next) {
            const new_listener = {
                onReceiveMessage(message, nextMessage) {
                  // after receiving response logic
                    savedMessage = message;
                    savedMessageNext = nextMessage;
                },
                onReceiveStatus(status, nextStatus) {
                  // after receiving status logic
                    savedMessageNext(savedMessage);
                    nextStatus(status);
                },
            };
            next(metadata, new_listener);
        },
    };
    return new grpc.InterceptingCall(nextCall(options), requester);
}

Chitti.add_call_interceptor(TestCallInterceptor); // adding globally
TestgrpcService.add_call_interceptor(TestCallInterceptor); // adding to specific service 
```

## 5. Custom Error Implementation
> RPCs provide a way to invoke remote methods as if they are locally available. However, this paradigm usally breaks down when it comes to handling errors. Chitti provides a local-like error handling model where service handlers throw custom exceptions and clients can handle those exceptions.

### Consider a service which throws CustomError1, CustomError2 errors
```protobuf
package testgrpc;

message CustomError1 {
	string content = 1;
	string id = 2;
}
message CustomError2 {
    string reason = 1;
    string status_code = 2;
}
```

### Classifying the protobuf message as Error
The above proto messages have to be classified as errors. This is done through `Error.enable` which takes an array of protobuf messages from `RPCImport` as the first argument and `Object` as the second one. To throw a specific error code set `code` as a key and the error code as value to the `Object`. If no code is specified, it takes `500` as the default value.

```js
import { Error } from 'chitti';
// Get  CError, CustomError from RPCImport
Error.enable([CustomError1,CustomError2]);// default code = 500
// or
Error.enable([CustomError1,CustomError2],{code:502});
// or
Error.enable([CustomError1],{code:502});
Error.enable([CustomError2],{code:504});
```

### Throwing an Error from handlers:
Note: Before throwing a protobuf message instance(error object), ensure that it is Error enabled otherwise the client will not recieve its as an [Error](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Error) object.
```js
class TestService extends TestgrpcService.Service {
    async hellogrpc(req) {
        const obj = { reason: 'error' };
        const errobj =  new CustomError2(obj);
        throw errobj;
    }
}
```

### Catching the Error at client end:

```js
(async () => {
    try {
        const response = await TestgrpcService.hellogrpc({ req_messsage: 'requesting hellogrpc' });
        console.log(response);
    }
    catch (error) {
        console.log(error);
        // =>
        //  CustomError2 {
        //  reason: ' error',
        //  code: 502 }
        console.log(error.reason); // => 'error'
        console.log(error.code); // => 502
        console.log(error.details); // => 'testgrpc.CustomError2        
    }
})();
```
