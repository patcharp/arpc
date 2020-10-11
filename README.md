# ARPC - More Effective Network Communication 

[![GoDoc][1]][2] [![MIT licensed][3]][4] [![Build Status][5]][6] [![Go Report Card][7]][8] [![Coverage Statusd][9]][10]

[1]: https://godoc.org/github.com/lesismal/arpc?status.svg
[2]: https://godoc.org/github.com/lesismal/arpc
[3]: https://img.shields.io/badge/license-MIT-blue.svg
[4]: LICENSE
[5]: https://travis-ci.org/lesismal/arpc.svg?branch=master
[6]: https://travis-ci.org/lesismal/arpc
[7]: https://goreportcard.com/badge/github.com/lesismal/arpc
[8]: https://goreportcard.com/report/github.com/lesismal/arpc
[9]: https://codecov.io/gh/lesismal/arpc/branch/master/graph/badge.svg
[10]: https://codecov.io/gh/lesismal/arpc




## Contents

- [ARPC - More Effective Network Communication](#arpc---more-effective-network-communication)
	- [Contents](#contents)
	- [Features](#features)
	- [Performance](#performance)
	- [Header Layout](#header-layout)
	- [Installation](#installation)
	- [Quick start](#quick-start)
	- [API Examples](#api-examples)
		- [Register Routers](#register-routers)
		- [Client Call, CallAsync, Notify](#client-call-callasync-notify)
		- [Server Call, CallAsync, Notify](#server-call-callasync-notify)
		- [Broadcast - Notify](#broadcast---notify)
		- [Async Response](#async-response)
		- [Handle New Connection](#handle-new-connection)
		- [Handle Disconnected](#handle-disconnected)
		- [Handle Client's send queue overstock](#handle-clients-send-queue-overstock)
		- [Custom Net Protocol](#custom-net-protocol)
		- [Custom Codec](#custom-codec)
		- [Custom Logger](#custom-logger)
		- [Custom operations before conn's recv and send](#custom-operations-before-conns-recv-and-send)
		- [Custom arpc.Client's Reader from wrapping net.Conn](#custom-arpcclients-reader-from-wrapping-netconn)
		- [Custom arpc.Client's send queue capacity](#custom-arpcclients-send-queue-capacity)
	- [PUB/SUB Examples](#pubsub-examples)

## Features
- [x] Two-Way Calling
- [x] Two-Way Notify
- [x] Sync and Async Calling
- [x] Sync and Async Response
- [x] Batch Write | Writev | net.Buffers 
- [x] Broadcast
- [x] Pub/Sub

| Pattern | Interactive Directions       | Description              |
| ------- | ---------------------------- | ------------------------ |
| call    | two-way:<br>c -> s<br>s -> c | request and response     |
| notify  | two-way:<br>c -> s<br>s -> c | request without response |


## Performance

- simple echo load testing

| Framework | Protocol        | Codec         | Configuration                                             | Connection Num | Goroutine Num | Qps     |
| --------- | --------------- | ------------- | --------------------------------------------------------- | -------------- | ------------- | ------- |
| arpc      | tcp/localhost   | encoding/json | os: VMWare Ubuntu 18.04<br>cpu: AMD 3500U 4c8t<br>mem: 2G | 8              | 10            | 80-100k |
| grpc      | http2/localhost | protobuf      | os: VMWare Ubuntu 18.04<br>cpu: AMD 3500U 4c8t<br>mem: 2G | 8              | 10            | 20-30k  |


## Header Layout

- LittleEndian

| bodylen | sequence | cmd    | isError | isAsync | methodlen | method               | body |
| ------- | -------- | ------ | ------- | ------- | --------- | -------------------- | ---- |
| 4 bytes | 8 bytes  | 1 byte | 1 byte  | 1 bytes | 1 bytes   | 0 or methodlen bytes | ...  |



## Installation

1. Get and install arpc

```sh
$ go get -u github.com/lesismal/arpc
```

2. Import in your code:

```go
import "github.com/lesismal/arpc"
```


## Quick start
 
- start a [server](https://github.com/lesismal/arpc/blob/master/examples/rpc_sync/server/server.go)

```go
package main

import "github.com/lesismal/arpc"

func main() {
	server := arpc.NewServer()

	// register router
	server.Handler.Handle("/echo", func(ctx *arpc.Context) {
		str := ""
		if err := ctx.Bind(&str); err == nil {
			ctx.Write(str)
		}
	})

	server.Run(":8888")
}
```

- start a [client](https://github.com/lesismal/arpc/blob/master/examples/rpc/client/client.go)

```go
package main

import (
	"log"
	"net"
	"time"

	"github.com/lesismal/arpc"
)

func main() {
	client, err := arpc.NewClient(func() (net.Conn, error) {
		return net.DialTimeout("tcp", "localhost:8888", time.Second*3)
	})
	if err != nil {
		panic(err)
	}

	client.Run()
	defer client.Stop()

	req := "hello"
	rsp := ""
	err = client.Call("/echo", &req, &rsp, time.Second*5)
	if err != nil {
		log.Fatalf("Call failed: %v", err)
	} else {
		log.Printf("Call Response: \"%v\"", rsp)
	}
}
```



## API Examples

### Register Routers

```golang
var handler arpc.Handler

// package
handler = arpc.DefaultHandler
// server
handler = server.Handler
// client
handler = client.Handler

handler.Handle("/route", func(ctx *arpc.Context) { ... })
handler.Handle("/route2", func(ctx *arpc.Context) { ... })
handler.Handle("method", func(ctx *arpc.Context) { ... })
```



### Client Call, CallAsync, Notify

1. Call (Block, with timeout/context)

```golang
request := &Echo{...}
response := &Echo{}
timeout := time.Second*5
err := client.Call("/call/echo", request, response, timeout)
// ctx, cancel := context.WithTimeout(context.Background(), time.Second)
// defer cancel()
// err := client.CallWith(ctx, "/call/echo", request, response)
```

2. CallAsync (Nonblock, with callback and timeout/context)

```golang
request := &Echo{...}

timeout := time.Second*5
err := client.CallAsync("/call/echo", request, func(ctx *arpc.Context) {
	response := &Echo{}
	ctx.Bind(response)
	...	
}, timeout)

// ctx, cancel := context.WithTimeout(context.Background(), time.Second)
// defer cancel()
// err := client.CallAsyncWith(ctx, "/call/echo", request, func(ctx *arpc.Context) {
// 	response := &Echo{}
// 	ctx.Bind(response)
// 	...	
// })
```

3. Notify (same as CallAsync with timeout/context, without callback)

```golang
data := &Notify{...}
client.Notify("/notify", data, time.Second)
// ctx, cancel := context.WithTimeout(context.Background(), time.Second)
// defer cancel()
// client.NotifyWith(ctx, "/notify", data)
```

### Server Call, CallAsync, Notify

1. Get client and keep it in your application

```golang
var client *arpc.Client
server.Handler.Handle("/route", func(ctx *arpc.Context) {
	client = ctx.Client
	// release client
	client.OnDisconnected(func(c *arpc.Client){
		client = nil
	})
})

go func() {
	for {
		time.Sleep(time.Second)
		if client != nil {
			client.Call(...)
			client.CallAsync(...)
			client.Notify(...)
		}
	}
}()
```

2. Then Call/CallAsync/Notify

- [See Previous](#client-call-callasync-notify)

### Broadcast - Notify

- for more details:	[**server**](https://github.com/lesismal/arpc/blob/master/examples/broadcast/server/server.go) [**client**](https://github.com/lesismal/arpc/blob/master/examples/broadcast/client/client.go)

```golang
var mux = sync.RWMutex{}
var clientMap = make(map[*arpc.Client]struct{})

func broadcast() {
	msg := arpc.NewMessage(arpc.CmdNotify, "/broadcast", fmt.Sprintf("broadcast msg %d", i), nil)
	mux.RLock()
	for client := range clientMap {
		client.PushMsg(msg, arpc.TimeZero)
	}
	mux.RUnlock()
}
```

### Async Response

```golang
var handler arpc.Handler

// package
handler = arpc.DefaultHandler
// server
handler = server.Handler
// client
handler = client.Handler

func asyncResponse(ctx *arpc.Context, data interface{}) {
	ctx.Write(data)
}

handler.Handle("/echo", func(ctx *arpc.Context) {
	req := ...
	err := ctx.Bind(req)
	if err == nil {
		go asyncResponse(ctx, req)
	}
})
```


### Handle New Connection

```golang
// package
arpc.DefaultHandler.HandleConnected(func(c *arpc.Client) {
	...
})

// server
svr := arpc.NewServer()
svr.Handler.HandleConnected(func(c *arpc.Client) {
	...
})

// client
client, err := arpc.NewClient(...)
client.Handler.HandleConnected(func(c *arpc.Client) {
	...
})
```

### Handle Disconnected

```golang
// package
arpc.DefaultHandler.HandleDisconnected(func(c *arpc.Client) {
	...
})

// server
svr := arpc.NewServer()
svr.Handler.HandleDisconnected(func(c *arpc.Client) {
	...
})

// client
client, err := arpc.NewClient(...)
client.Handler.HandleDisconnected(func(c *arpc.Client) {
	...
})
```

### Handle Client's send queue overstock

```golang
// package
arpc.DefaultHandler.HandleOverstock(func(c *arpc.Client) {
	...
})

// server
svr := arpc.NewServer()
svr.Handler.HandleOverstock(func(c *arpc.Client) {
	...
})

// client
client, err := arpc.NewClient(...)
client.Handler.HandleOverstock(func(c *arpc.Client) {
	...
})
```

### Custom Net Protocol

```golang
// server
var ln net.Listener = ...
svr := arpc.NewServer()
svr.Serve(ln)

// client
dialer := func() (net.Conn, error) { 
	return ... 
}
client, err := arpc.NewClient(dialer)
```
 
### Custom Codec

```golang
var codec arpc.Codec = ...

// package
arpc.DefaultCodec = codec

// server
svr := arpc.NewServer()
svr.Codec = codec

// client
client, err := arpc.NewClient(...)
client.Codec = codec
```

### Custom Logger

```golang
var logger arpc.Logger = ...
arpc.SetLogger(logger) // arpc.DefaultLogger = logger
``` 

### Custom operations before conn's recv and send

```golang
arpc.DefaultHandler.BeforeRecv(func(conn net.Conn) error) {
	// ...
})

arpc.DefaultHandler.BeforeSend(func(conn net.Conn) error) {
	// ...
})
```

### Custom arpc.Client's Reader from wrapping net.Conn 

```golang
arpc.DefaultHandler.SetReaderWrapper(func(conn net.Conn) io.Reader) {
	// ...
})
```

### Custom arpc.Client's send queue capacity 

```golang
arpc.DefaultHandler.SetSendQueueSize(4096)
```

## PUB/SUB Examples

- start a server
```golang
import "github.com/lesismal/arpc/pubsub"

var (
	address = "localhost:8888"

	password = "123qwe"

	topicName = "Broadcast"
)

func main() {
	s := pubsub.NewServer()
	s.Password = password

	// server publish to all clients
	go func() {
		for i := 0; true; i++ {
			time.Sleep(time.Second)
			s.Publish(topicName, fmt.Sprintf("message from server %v", i))
		}
	}()

	s.Run(address)
}
```

- start a subscribe client
```golang
import "github.com/lesismal/arpc"
import "github.com/lesismal/arpc/pubsub"

var (
	address = "localhost:8888"

	password = "123qwe"

	topicName = "Broadcast"
)

func onTopic(topic *pubsub.Topic) {
	arpc.DefaultLogger.Info("[OnTopic] [%v] \"%v\", [%v]",
		topic.Name,
		string(topic.Data),
		time.Unix(topic.Timestamp/1000000000, topic.Timestamp%1000000000).Format("2006-01-02 15:04:05.000"))
}

func main() {
	client, err := pubsub.NewClient(func() (net.Conn, error) {
		return net.DialTimeout("tcp", address, time.Second*3)
	})
	if err != nil {
		panic(err)
	}
	client.Password = password
	client.Run()

	// authentication
	err = client.Authenticate()
	if err != nil {
		panic(err)
	}

	// subscribe topic
	if err := client.Subscribe(topicName, onTopic, time.Second); err != nil {
		panic(err)
	}

	<-make(chan int)
}
```

- start a publish client
```golang
import "github.com/lesismal/arpc/pubsub"

var (
	address = "localhost:8888"

	password = "123qwe"

	topicName = "Broadcast"
)

func main() {
	client, err := pubsub.NewClient(func() (net.Conn, error) {
		return net.DialTimeout("tcp", address, time.Second*3)
	})
	if err != nil {
		panic(err)
	}
	client.Password = password
	client.Run()

	// authentication
	err = client.Authenticate()
	if err != nil {
		panic(err)
	}

	for i := 0; true; i++ {
		if i%5 == 0 {
			// publish msg to all clients
			client.Publish(topicName, fmt.Sprintf("message from client %d", i), time.Second)
		} else {
			// publish msg to only one client
			client.PublishToOne(topicName, fmt.Sprintf("message from client %d", i), time.Second)
		}
		time.Sleep(time.Second)
	}
}
```


