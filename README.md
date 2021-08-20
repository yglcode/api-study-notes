## Study Notes: Small(Deep) APIs with Typeless(Opaque) Data ##
### _diff between Application API and System API; Go net/rpc is impressively "small"_ ###

### **1. Small(Narrow/Deep) API with typeless/Opaque Data.** ###

   Recently i watched the video of John Ousterhout's "A Philosophy of Software Design" [youtube link](https://www.youtube.com/watch?v=bmSAYlu0NcY). He teaches the importance of small/deep API/interfaces for reducing software complexity. The prime example he give is the Unix file IO interface (open/close/read/write/lseek).

   As a metaphor of the situation, suppose we have a complicated twisted rope of wires, how we can get a "minimal" interface to all wires? The solution is a "cut" perpendicular to the rope, or an interface/API "orthogonal" to normal flows.

   Applications have different data file formats, such as pdf, word, jpeg, mpeg4, etc. Some files may contains a header, multiple records and possibly a tail. An OOP design will have classes for header/records/tail, and methods for reading/writing these parts.

   Many devices have different connection "address" strings, config data, and commands for control and data exchange. OOP design will have classes for addess, configs and devices, and methods for sending control and data exchange commands.

   Unix file IO api "cut" through all the complexity, unifying them with a small set of methods:
```c
   int open(const char *pathname, int flags, mode_t mode);
   ssize_t read(int fd, void* buf, size_t count);
   ssize_t write(int fd, const void* buf, size_t count);
   ...
```
   Looking at the above methods, we can see a common theme:
        * use string pathname to identify resource/object in a hierarchical namespace.
        * use a small set of operations to manipulate the resource: open/close/read/write/seek
        * the api methods (read, write) use "typeless" data (void *), plain byte stream; all app specific data type info (pdf,jpeg,header,record...) are removed.

   If we want our api to be "strongly-typed", keep app data type info in api, the abstraction will be bogged down by diff use cases. An abstraction promotes a particular view of system: focus on a particular set of "essential" concepts and hide others as "minor" details.

   How do we hide details in programming?
        * don't mention them (in api signatures).
        * make them "typeless" (ie. can be anything): use void* in C/C++, or empty interface{} in Go.

   Another prime example of small/deep API which exhibits the above common theme is HTTP protocol:
      * resources are identified by string URI/URLs in hierarchical namespace
      * resources are manipulated thru a small set of methods (or verbs): Get/Put/Post/Delete/Patch
      * the api methods use typeless data: request/response body are binary blobs (octet*), which are decoded based on Content-Type and Content-Encoding headers.
      * they are "deep": covering all the use cases of web based programming (RESTful), retrieving/updating database, controlling remote services and devices,....

### **2. Difference between application APIs and system apis.** ###

   For application APIs, people prefer rich data types for application domain to provide conceptual model, and strongly-typed methods to guard against programming errors. This is in contrast to the above mentioned file IO and http apis, which we may call the "system" level apis, for the separation.

   So for convenience of building applications, people will wrap the small/"typeless" system apis with strongly typed SDKs.

   And AWS provides SDKs for all its services, which expose language idiomatic, strongly typed data structs and methods to access its services; while internally SDKs will validate and encode the api calls into HTTP messages to its service endpoints, and decode response messages as results to return the api calls.

### **3. Go net/rpc is impressively "small".** ###

   Go net/rpc [api doc](https://pkg.go.dev/net/rpc@go1.17) is small in every sense. Implementation wise, its main client/server code combined is around 900 lines. Api wise, besides various ways to set up client-server connections, there are only two methods for remote method call:
```go
   // for synchronous call
   func (cli *Client) Call(serviceMethod string, args interface{}, reply interface{}) error
   // for asynchronous call
   func (cli *Client) Go(serviceMethod string, args interface{}, reply interface{}, done chan *Call) *Call
```
   However, net/rpc is not a toy, successfully used in real production in many places: hashicorp, minio, crockroachdb, etc., and exhibits good request-response performance [cockroachdb benchmark](https://github.com/cockroachdb/rpc-bench).

   For gRPC based services, we will first define application specific protobuf schema files and generate stub/skeleton code, which will provide domain specific rich data types and strongly typed api methods.

   Compared to gRPC, net/rpc is more similar to the above mentioned "system api": 
      * remote methods/functions exposed at server are organized into hierachical namespace, which can be as simple as "serviceName.methodName". This "pathname" is the 1st argument of Call()/Go(), what client use to identify the remote function.   
      * If we treat "remote functions" as resources, what we can do to them? We can only invoke them. So the set of operations we can apply to them are different ways to invoke remote functions:
         . synchronous invoke (Call): invoke remote function and wait for result.
         . asynchronous invoke (Go): invoke remote funtions, don't wait and return right away with a result-pending "done" channel (or "future" in other language); continue with other work and later retrieve the result (or error) thru "done" channel (or "future").
         . one-way call (FireAndForget): ?
      * api operations (Call,Go) only use "typeless" data: "args interface{}. reply interface{}". The important details in normal function signature,  number/types of arguments and result, are all removed and replaced with empty interface{}, that allows this api to fit the different data structures required in various applications.

   Of course similar to other system api such as http, in real production, people often wrap strongly-typed SDK over net/rpc to support application code.

