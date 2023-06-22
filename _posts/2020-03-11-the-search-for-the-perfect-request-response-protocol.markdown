---
date: 2020-03-11 17:14:06
title: Request-Response Olympics — The Search For The Perfect Request-Response Protocol
description: The Search For The Perfect Request-Response Protocol
image: images/posts/the-search-for-the-perfect-request-response-protocol/the-search-for-the-perfect-request-response-protocol.webp
categories: [networking, microservices]
tags: [event-driven-design, request-response, http, grpc, redis, tcp]
---
Services tend to interact with each other. It may be microservices collaboratively processing your data, separate systems notifying each other on various events, or simply data propagating from one place to another. There are countless examples, all relying on programs sending messages to one another.

The flow of communication between systems may roughly be divided into two groups:

-   **Request-Response Messaging** - in which the client awaits the server response and then uses it in the context of the original request.
-   **event driven design** - in which messages are received on one end but not necessarily yield a single response back to the original sender.

In the world of big data processing we can usually design our systems to work with event driven design. It's simpler to maintain, it's much cheaper, it's more robust but most importantly, there are countless solutions for this domain. From messaging brokers like RabbitMQ, SQS and many more, through streaming platforms like Kafka, to Reactive Streams, Actor Model and gRPC streams. From open source solutions to managed solutions to countless online discussions. If you can go for a design that leverages this kind of communication, you probably should.

But sometimes you can't. In some designs you're going to have to perform your tasks in real time, split between multiple services, each performing its own duties. In such a case you'll probably need the gateway service to send request to one or more other services, await their response and eventually compose a response and reply back to the user. 😓

This kind of processing requires a request-response mechanism, something that will allow you to send requests to other services and await their response, but wait...it sounds, oh, so familiar 🤔 it's just what HTTP does. Well, as it turns out HTTP is what we usually use for this kind of requirement, but it's probably not the best choice to make, to say the least.

What if we can make a contest between some request-response messaging alternatives, discuss each one, show a quick implementation in Go and then give it a final score??? Oh wait, we can! 💪 👩 And this is exactly what we're going to do. We're going to look for the following:

-   **Performance** - we will test response times of each protocol for a duration of 60 seconds at a rate of 100 requests per second.
-   **Ease of use** - what you'll need to achieve a production ready client and server.
-   **Scaling** - load balancing options, limitations, etc.

The full benchmark repository can be found in Github (<https://github.com/PerimeterX/rpc-protocol-benchmark>). Each section below contains an example snippet taken from it. Feel free to clone it and run the tests yourself.

Ready...Set...GO! 🏋‍♀️

## HTTP

Oh, the queen of the internet, slayer of protocols. HTTP is probably powering most of the internet. It's simple, probably every software engineer knows it and has many implementations in every language. The problem is it's also the least performant one.

I won't show an example client-server in HTTP since we've all seen too many of those, although you can feel free to view the [benchmark client-server implementation](https://github.com/PerimeterX/rpc-protocol-benchmark/blob/master/http.go) just to make sure the contest is fair.

![HTTP Response Duration](/images/posts/the-search-for-the-perfect-request-response-protocol/rpc-olympics-http-response-duration.webp)

HTTP Response Duration

- **Performance** (💔💔💔) - As you'll see further below, HTTP was the slowest in all percentiles.
- **Ease of use** (🤙🤙🤙) - Well, this is probably the main advantage. Tons of frameworks for both client and server, just choose the one you like best. You've probably used it before, so it's also a plus.
- **Scaling** (🤷‍♀️🤷‍♀️🤷‍♀️) - A simple load balancer will do the trick. You can go for a managed solution from your cloud provider or simply use an open solution like Envoy, Nginx or HAProxy. This means harming response times even more, but at least a simple solution is an option (as opposed to some of the other protocols).

## gRPC

Born and raised in Google, powered by [Protobuf](https://github.com/protocolbuffers/protobuf) and HTTP2 designed specifically for these kinds of needs, gRPC sounds like the real deal.

gRPC is schema based, so you first need to define your service, its endpoints and the request and response schema of each endpoint in a protobuf `proto` file.

```proto
// defining gRPC Service

syntax = "proto3";
option go_package = "main";

service GrpcService {
    rpc Greet (GreetRequest) returns (GreetResponse) {}
}

message GreetRequest {
    string name = 1;
}

message GreetResponse {
    string greeting = 1;
}
```

Then protobuf can simply generate a template code for client and server in many languages. Make sure you install protobuf and the plugin for the language you work with.

`protoc - go_out=plugins=grpc:. grpc.proto`

Then the server looks like this

```go
// grpc server

package main

import (
	"context"
	"fmt"
	"google.golang.org/grpc"
	"net"
)

var s *grpc.Server

func initServer() {
	lis, e := net.Listen("tcp", ":8080")
	if e != nil {
		panic(fmt.Sprintf("could not create grpc listener: %v", e.Error()))
	}
	s = grpc.NewServer()
	service := &grpcService{}
	RegisterGrpcServiceServer(s, service)
	go func() {
		if e := s.Serve(lis); e != nil {
			panic(fmt.Sprintf("could not start grpc server: %v", e.Error()))
		}
	}()
}


func closeServer() {
	s.Stop()
}

type grpcService struct{}

func (g *grpcService) Greet(_ context.Context, req *GreetRequest) (*GreetResponse, error) {
	return &GreetResponse{Greeting: fmt.Sprintf("hello, %s", req.Name)}, nil
}
```

And the client looks like this

```go
// grpc client

package main

import (
	"context"
	"fmt"
	"google.golang.org/grpc"
)

var (
	conn *grpc.ClientConn
	c    GrpcServiceClient
)

func initClient() {
	var e error
	conn, e = grpc.Dial("localhost:8080", grpc.WithInsecure(), grpc.WithBlock())
	if e != nil {
		panic(fmt.Sprintf("could not start grpc client: %v", e.Error()))
	}
	c = NewGrpcServiceClient(conn)
}

func doRPC(name string) {
	r, e := c.Greet(context.Background(), &GreetRequest{Name: name})
	if e != nil {
		panic(fmt.Sprintf("grpc error: %s", e.Error()))
	}
	if r.Greeting != fmt.Sprintf("hello, %s", name) {
		panic(fmt.Sprintf("wrong grpc answer: %s", r.Greeting))
	}
}

func closeClient() {
	if e := conn.Close(); e != nil {
		panic(fmt.Sprintf("could not close grpc client: %v", e.Error()))
	}
}
```

So...how did it do? Well, better than HTTP, but not good enough.

![gRPC Response Duration](/images/posts/the-search-for-the-perfect-request-response-protocol/rpc-olympics-grpc-response-duration.webp)

- **Performance** (🤷‍♀️🤷‍♀️🤷‍♀️) - It heavily depends on your use case. Protobuf is much faster to encode and decode comparing to HTTP's form data or to JSON. More importantly it sends much smaller (binary) messages on the wire. If you need to pass large, schema compliant messages this will give gRPC a big plus.
But then again you can use Protobuf to encode and decode HTTP messages as well, so if we ignore the message aspect and focus on the actual protocol, you'd probably expect better results.
- **Ease of use** (😲😲😲) - Well, this is very subjective. Personally, I like things to be statically defined, and I don't mind installing Protobuf once if it's going to serve me well. But this is obviously more work than HTTP.
- **Scaling** (😓💪) - This one is trickier. A simple load balancer will not help much since connections may be kept open. Basically, you have two main options here:
  - [Envoy gRPC mode](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/other_protocols/grpc), which will hurt performance a bit, but should keep things simple and clean.
  - Client side load balancing. This is the preferred option if you want to remain fast and effective. This could mean, for example, using a service discovery to report all available servers to all clients, and let the client round-robin each request to the next server.

## gRPC Stream

gRPC works in four different modes. Simple gRPC (which we've seen above), server side stream where the server sends multiple messages upon a single message from the client, client side stream where the client sends multiple messages and eventually receives a single response and bidirectional stream, where the client establishes connection and then basically everyone does whatever they want.

This means multiple messages going back and forth on the same connection. This is classic for event driven design as discussed in the intro, but 💡...why not use it for Request-Response? We can mark each request with a unique identifier and then have the server send it back on each response. Why not? Well, because it's complicated and risky. But it's very, very fast.

![gRPC Stream Response Duration](/images/posts/the-search-for-the-perfect-request-response-protocol/rpc-olympics-grpc-stream-response-duration.webp)

So how did I do it? Let's start with the `proto` file.

```proto
// defining grpc bidirectional stream service

syntax = "proto3";
option go_package = "main";

service GrpcStreamService {
    rpc Greet (stream StreamGreetRequest) returns (stream StreamGreetResponse) {}
}

message StreamGreetRequest {
    string id = 1;
    string name = 2;
}

message StreamGreetResponse {
    string id = 1;
    string greeting = 2;
}
```

Then the server, notice how it echoes the id back in the response.

```go
// grpc stream server

package main

import (
	"fmt"
	"google.golang.org/grpc"
	"io"
	"net"
)

var s *grpc.Server

func initServer() {
	lis, e := net.Listen("tcp", ":8080")
	if e != nil {
		panic(fmt.Sprintf("could not create grpc stream listener: %v", e.Error()))
	}
	s = grpc.NewServer()
	service := &grpcStreamService{}
	RegisterGrpcStreamServiceServer(s, service)
	go func() {
		if e := s.Serve(lis); e != nil {
			panic(fmt.Sprintf("could not start grpc stream server: %v", e.Error()))
		}
	}()
}

func closeServer() {
	s.Stop()
}

type grpcStreamService struct{}

func (g *grpcStreamService) Greet(stream GrpcStreamService_GreetServer) error {
	for {
		in, e := stream.Recv()
		if e == io.EOF {
			return nil
		}
		if e != nil {
			return e
		}
		if e := stream.Send(&StreamGreetResponse{Id: in.Id, Greeting: fmt.Sprintf("hello, %s", in.Name)}); e != nil {
			panic(fmt.Sprintf("grpc stream send response error: %s", e.Error()))
		}
	}
}
```

And lastly, the client. Most of the work is done here. Notice how it holds a collection of callbacks mapped by the request id, then when a response is received the pending callback is called.

```go
// grpc stream client

package main

import (
"context"
"fmt"
"google.golang.org/grpc"
	"sync"
)

var (
	conn      *grpc.ClientConn
	c         GrpcStreamServiceClient
	lock      sync.Mutex
	callbacks map[string]func(*StreamGreetResponse)
	stream    GrpcStreamService_GreetClient
)

func initClient() {
	callbacks = make(map[string]func(*StreamGreetResponse))
	var e error
	if conn, e = grpc.Dial("localhost:8080", grpc.WithInsecure(), grpc.WithBlock()); e != nil {
		panic(fmt.Sprintf("could not start grpc stream client: %v", e.Error()))
	}
	c = NewGrpcStreamServiceClient(conn)
	if stream, e = c.Greet(context.Background()); e != nil {
		panic(fmt.Sprintf("count not create grpc client stream: %v", e.Error()))
	}
	go func() {
		for {
			in, e := stream.Recv()
			if e != nil {
				return
			}
			lock.Lock()
			callback, exists := callbacks[in.Id]
			lock.Unlock()
			if !exists {
				panic(fmt.Sprintf("grpc stream client callback %s does not exist", in.Id))
			}
			callback(in)
		}
	}()
}

func doRPC(name string) {
	wg := sync.WaitGroup{}
	wg.Add(1)
	lock.Lock()
	callbacks[name] = func(response *StreamGreetResponse) {
		if response.Greeting != fmt.Sprintf("hello, %s", name) {
			panic(fmt.Sprintf("wrong grpc stream answer: %s", response.Greeting))
		}
		lock.Lock()
		defer lock.Unlock()
		delete(callbacks, name)
		wg.Done()
	}
	lock.Unlock()
	if e := stream.Send(&StreamGreetRequest{Id: name, Name: name}); e != nil {
		panic(fmt.Sprintf("grpc error: %s", e.Error()))
	}
	wg.Wait()
}

func closeClient() {
	if e := stream.CloseSend(); e != nil {
		panic(fmt.Sprintf("could not close grpc stream: %v", e.Error()))
	}
	if e := conn.Close(); e != nil {
		panic(fmt.Sprintf("could not close grpc stream client: %v", e.Error()))
	}
}
```

- **Performance** (🤯🤯🤯) - This is fast. How fast? Median requests took 12% of median HTTP time to return. Basically, in an average use case you could probably accelerate your HTTP architecture by more than 8 times.
- **Ease of use** (👎👎👎) - Let's be honest here, you shouldn't go to production with this snippet. Handling these callbacks raise a ton of risks and edge cases to handle. For starters, you must set timeouts and remove zombie callbacks, in general you may run into memory leaks and non responding servers.

## Redis Compatible Server

You've probably heard about Redis, the [top ranked key-value database](https://db-engines.com/en/ranking) for years now. Speed and simplicity are probably main contributors to it. Redis clients interact with the server using a pool of TCP connections with a thin layer of data encoding. The client simply sends a list of arguments, the first of which is the name of the command to be done.

In order to leverage the above into a generic client server, you simply need to implement a Redis compatible server. In Go there's a neat library called [redcon](https://github.com/tidwall/redcon) which does exactly that. So how does it look?

```go
// redis compatible server


package main

import (
	"fmt"
	"github.com/tidwall/redcon"
	"net"
	"strings"
)

var lis net.Listener

func initServer() {
	var e error
	if lis, e = net.Listen("tcp", ":8080"); e != nil {
		panic(fmt.Sprintf("coult not start redcon listener: %s", e.Error()))
	}
	go func() {
		if e := redcon.Serve(lis, handler, accept, closed); e != nil {
			panic(fmt.Sprintf("could not start redcon server: %s", e.Error()))
		}
	}()
}

func closeServer() {
	if e := lis.Close(); e != nil {
		panic(fmt.Sprintf("could not close redcon server: %s", e.Error()))
	}
}

func handler(conn redcon.Conn, cmd redcon.Command) {
	cmdName := strings.ToLower(string(cmd.Args[0]))
	if cmdName != "greet" {
		conn.WriteError(fmt.Sprintf("invalid cmd %s", cmdName))
		return
	}
	name := strings.ToLower(string(cmd.Args[1]))
	conn.WriteString(fmt.Sprintf("hello, %s", name))
}

func accept(_ redcon.Conn) bool {
	return true
}

func closed(_ redcon.Conn, _ error) {}
```

```go
// redis client

package main

import (
	"fmt"
	"github.com/go-redis/redis"
)

var client *redis.Client

func initClient() {
	client = redis.NewClient(&redis.Options{Addr: "localhost:8080"})
}

func doRPC(name string) {
	res, e := client.Do("greet", name).Result()
	if e != nil {
		panic(fmt.Sprintf("error sending redcon request: %s", e.Error()))
	}
	data, ok := res.(string)
	if !ok {
		panic(fmt.Sprintf("unexpected redcon response type"))
	}
	if data != fmt.Sprintf("hello, %s", name) {
		panic(fmt.Sprintf("wrong redcon answer: %s", data))
	}
}

func closeClient() {
	if e := client.Close(); e != nil {
		panic(fmt.Sprintf("could not close redcon client: %s", e.Error()))
	}
}
```

Notice the `cmdName != "greet"` part of the server. This is the actual routing of requests. You should implement a simple router which triggers a matching handler for each command name.
So how fast can it go?

![Redcon Response Duration](/images/posts/the-search-for-the-perfect-request-response-protocol/rpc-olympics-redcon-response-duration.webp)

- **Performance** (👍👍👍) - Well, it did 5.5 times faster than HTTP on median requests.
- **Ease of use** (🤜🤛) - The tricky part is creating a server compatible to Redis protocol. If you go into the [Go library implementation](https://github.com/tidwall/redcon) you'll find 3 files implementing the entire thing. If you already have such a library in a language of your choosing, well then, you're a few lines of code from implementing a production ready server.
And what about the client? Well, since it's just a regular Redis client you already have tons of them, and in every language you like. You can communicate with a single node server or a cluster, you can configure the connection pool size, timeout and probably anything else you can think of.
- **Scaling** (😓💪) - Same as in gRPC, you can either go with:
  - [Envoy Redis mode](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/other_protocols/redis), which will hurt performance a bit, but should keep things simple and clean.
  - Client side load balancing. This is the preferred option if you want to remain fast and effective. This could mean, for example, using a service discovery to report all available servers to all clients, and let the client round-robin each request to the next server.

## Plain TCP Server

Let's go as low-level as we can with this. The downside here should be that it's the most code to write, right? Well, not *that* much, apparently.

```go
// tcp server

package main

import (
	"bufio"
	"fmt"
	"io"
	"net"
	"strings"
)

var lis net.Listener

func initServer() {
	var e error
	if lis, e = net.Listen("tcp", ":8080"); e != nil {
		panic(fmt.Sprintf("could not create tcp listener: %s", e.Error()))
	}
	go func() {
		for {
			conn, e := lis.Accept()
			if e != nil {
				if strings.Contains(e.Error(), "use of closed network connection") {
					return
				}
				panic(fmt.Sprintf("could not accept tcp connection: %s", e.Error()))
			}
			reader := bufio.NewReader(conn)
			go func() {
				for {
					data, e := reader.ReadString('n')
					if e == io.EOF {
						return
					}
					if e != nil {
						panic(fmt.Sprintf("could not read tcp request: %s", e.Error()))
					}
					if _, e := conn.Write([]byte(fmt.Sprintf("hello, %s", data))); e != nil {
						panic(fmt.Sprintf("could not write tcp response: %s", e.Error()))
					}
				}
			}()
		}
	}()
}

func closeServer() {
	if e := lis.Close(); e != nil {
		panic(fmt.Sprintf("could not close tcp listener: %s", e.Error()))
	}
}
```

```go
// tcp client

package main

import (
	"bufio"
	"fmt"
	"net"
)

const poolSize = 15

var clientPool *tcpConnectionPool

func initClient() {
	clientPool = newTCPConnectionPool()
}

func doRPC(name string) {
	c := clientPool.acquire()
	defer clientPool.release(c)
	if _, e := fmt.Fprintf(c, fmt.Sprintf("%sn", name)); e != nil {
		panic(fmt.Sprintf("could not send tcp request: %s", e.Error()))
	}
	reader := bufio.NewReader(c)
	data, e := reader.ReadString('n')
	if e != nil {
		panic(fmt.Sprintf("could not read tcp response: %s", e.Error()))
	}
	if data != fmt.Sprintf("hello, %sn", name) {
		panic(fmt.Sprintf("wrong tcp answer: %s", data))
	}
}

func closeClient() {
	clientPool.close()
}

type tcpConnectionPool struct {
	ch chan net.Conn
}

func newTCPConnectionPool() *tcpConnectionPool {
	pool := &tcpConnectionPool{ch: make(chan net.Conn, poolSize)}
	for i := 0; i < poolSize; i++ {
		conn, e := net.Dial("tcp", "localhost:8080")
		if e != nil {
			panic(fmt.Sprintf("could not start tcp client: %s", e.Error()))
		}
		pool.ch <- conn
	}
	return pool
}

func (t *tcpConnectionPool) acquire() net.Conn {
	return <-t.ch
}

func (t *tcpConnectionPool) release(c net.Conn) {
	t.ch <- c
}

func (t *tcpConnectionPool) close() {
	for i := 0; i < poolSize; i++ {
		conn := <-t.ch
		if e := conn.Close(); e != nil {
			panic(fmt.Sprintf("could not close tcp client: %s", e.Error()))
		}
	}
}
```

Is it ready to go to production? Well, no, for example:

-   You still need to create some protocol for routing of different requests to different handlers.
-   The client connection pool is very basic, although you could probably find implementations in any language.
-   What about request timeouts? Error handling? There can still be many edge cases to handle.
-   What about encoding of requests? How do you mark the end of a single message?

But the speed? Oh, the speed.

![TCP Response Duration](/images/posts/the-search-for-the-perfect-request-response-protocol/rpc-olympics-tcp-response-duration.webp)

- **Performance** (💋💋💋) - It went 10 times faster than HTTP on median requests.
- **Ease of use** (🙈🙉🙊) - Well, there are still many open questions for this to be ready to be shipped to production, but the basics are very simple and straightforward.
- **Scaling** (🥴🤒) - Here you're left with client side load balancing, it's your only reliable option, but it will surely keep things fast and performant.

## And The Winner Is... 🤽‍♀️🤼‍♀️🏅🏃‍♀️

![Request-Response Duration of Median Requests](/images/posts/the-search-for-the-perfect-request-response-protocol/rpc-olympics-median.webp)

![Request-Response Duration of Fastest Requests](/images/posts/the-search-for-the-perfect-request-response-protocol/rpc-olympics-fastest.webp)

![Request-Response Duration of 90th Pecentile Requests](/images/posts/the-search-for-the-perfect-request-response-protocol/rpc-olympics-90.webp)

![Request-Response Duration of Slowest Requests](/images/posts/the-search-for-the-perfect-request-response-protocol/rpc-olympics-slowest.webp)

## What About Different Use Cases?

In production, you'd probably need more than hello world. What if your messages are smaller or bigger? What if they are dynamic or static? What if you send unstructured data like images or other media?

Request-Response messaging protocols are simply means to transmit bytes on the wire. The benchmark performed here merely tests how fast data moves from side to side with fixed clients and servers. Your use case should dictate which encoding method you choose - if you need to send schema-less objects with changing fields and types JSON might be nice and cozy. If you send large objects with many static fields Protobuf will be blazing fast. You should decide how to encode messages based on your use case and then use it on top of your chosen messaging protocol, make it two separate discussions. Hey, even [gRPC can support JSON](https://grpc.io/blog/grpc-with-json/) if you ask it nicely.

## Conclusion

On a personal take, I think a Redis compatible server is the most balanced choice between fast and easy to maintain. Obviously, it depends much on your needs. If performance is not at the top of your priorities HTTP will have a lot to offer. If you're after robustness and schema correctness gRPC might be your choice. If you feel risky and experimental try implementing your own protocol on top of plain TCP. I recently switched from gRPC to Redis protocol in production on behalf of performance and got a huge improvement.

As a rule of thumb - let your needs dictate the messaging protocol and your use case the message encoding method.

## What Have I Missed?

Have you used other Request-Response protocols or solutions? Have you implemented your own? Please, let me know about it! Comment me suggestions for things I've missed.