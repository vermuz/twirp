# Getting Started

Let's skip Hello World, we'll make a `Haberdasher` service instead.

The `Haberdasher` service makes hats. It has only one RPC method `MakeHat` to make a new hat of a
particular size.

## Create Proto File

> Define your service in a **Proto** file:

```proto
// haberdasher.proto

syntax = "proto3";

package twirp.example.haberdasher;
option go_package = "haberdasher";

// Haberdasher service makes hats for clients.
service Haberdasher {
  // MakeHat produces a hat of mysterious, randomly-selected color!
  rpc MakeHat(Size) returns (Hat);
}

// Size of a Hat, in inches.
message Size {
  int32 inches = 1; // must be > 0
}

// A Hat is a piece of headwear made by a Haberdasher.
message Hat {
  int32 inches = 1;
  string color = 2; // anything but "invisible"
  string name = 3; // i.e. "bowler"
}
```

Twirp uses a **Proto** file to generate your service for you. See
[Protobuf](https://developers.google.com/protocol-buffers/) for more information on protobufs.

It's a good idea to add comments on your Protobuf file. These files can work as the main
documentation of your API.

## Auto-Generate Code

```bash
$ protoc --proto_path=$GOPATH/src:. --twirp_out=. --go_out=. ./hello_world.proto
```

> Twirp will use the above sh command to auto-generate the following files:

```
/
  haberdasher.pb.go    # auto-generated by protoc-gen-go (for protobuf serialization)
  haberdasher.proto    # original protobuf definition (you created this)
  haberdasher.twirp.go # auto-generated by protoc-gen-twirp (servers, clients and interfaces)
```

Run the `protoc` command pointed at your `.proto` files to generate `{service_name}.pb.go` and
`{service_name}.twirp.go` in the same directory as your `.proto` file.

<div class="clear"></div>

```golang
// haberdasher.twirp.go

// A Haberdasher makes hats for clients.
type Haberdasher interface {
    // MakeHat produces a hat of mysterious, randomly-selected color!
    MakeHat(context.Context, *Size) (*Hat, error)
}
```

Inside `haberdasher.twirp.go` you should find the following interface along with code to 
instantiate clients and servers.


## Implement the Server

> Our implementation of the server below:

```golang
// server.go

package haberdasherserver

import (
    "math/rand"
    "golang.org/x/net/context"
    "code.justin.tv/common/twirp"
    pb "code.justin.tv/twirpexample/rpc/haberdasher"
)

// Server implements the Haberdasher service
type Server struct {}

func (s *Server) MakeHat(ctx context.Context, size *pb.Size) (hat *pb.Hat, error) {
    if size.Inches <= 0 {
        return nil, twirp.InvalidArgumentError("inches", "I can't make a hat that small!")
    }
    return &pb.Hat{
        Inches:  size.Inches,
        Color: []string{"white", "black", "brown", "red", "blue"}[rand.Intn(4)],
        Name:  []string{"bowler", "baseball cap", "top hat", "derby"}[rand.Intn(3)],
    }, nil
}
```

Our server implementation `Server` must implement all methods in the interface generated by Twirp. 
Here we implement the function `MakeHat` that takes in `size *pb.Size` and outputs a new `&pb.Hat{}`.

Twirp servers can handle both
[JSON and Protobuf](https://github.com/twitchtv/twirp/wiki/Protobuf-and-JSON)
requests, distinguishing them by their Content-Type header. Twirp servers can also be configured with
[Server Hooks](https://github.com/twitchtv/twirp/wiki/Server-Hooks).

<div class="clear"></div>

> Auto-generated server constructor in `haberdasher.twirp.go` below:

```golang
func NewHaberdasherServer(svc Haberdasher, hooks *twirp.ServerHooks, ctxSrc ContextSource) http.Handler
```

Then, to serve it over HTTP, use the auto-generated server constructor `New{{Service}}Server`.
For Haberdasher this is `NewHaberdasherServer`.

<div class="clear"></div>

> Add the following main function to `server.go`:

```golang
// server.go

func main() {
    server := &haberdasherserver.Server{} // implements Haberdasher interface
    twirpHandler := haberdasher.NewHaberdasherServer(server, nil, nil)

    http.ListenAndServe(":8080", twirpHandler)
}
```

This constructor wraps your interface implementation as an `http.Handler`, so you can start serving
requests with `http.ListenAndServe()`.

<div class="clear"></div>

> An example of serving the haberdash server using Goji:

```golang
// server.go

func main() {
    server := &haberdasherserver.Server{} // implements Haberdasher interface
    twirpHandler := haberdasher.NewHaberdasherServer(server, nil, nil)

    mux := goji.NewMux()
    mux.Handle(pat.Post(haberdasher.NewHaberdasherPathPrefix+"*"), twirpHandler)
    // mux.Handle other things like health checks ...
    http.ListenAndServe("localhost:8000", mux)
}
```

You can use any type of **mux** to serve the server (eg. Goji).

## Using the Client

> To use the `Haberdasher` service from another Go project, just import the auto-generated client
> and then use it like this:

```golang
client := haberdasher.NewHaberdasherProtobufClient("http://localhost:8000", &http.Client{})

hat, err = client.MakeHat(context.Background(), &example.Size{Inches: 12})
if err != nil {
    if twerr, ok := err.(twirp.Error) {
        switch twerr.Code() {
        case twirp.InvalidArgument:
            fmt.Println("Oh no "+twirp.Msg())
        default:
            fmt.Println(twerr.Error())
        }
    }
    return
}
```

Clients are automatically generated. YAY!

For each service, there are 2 client constructors:

* `New{{Service}}ProtobufClient` for Protobuf requests.
* `New{{Service}}JSONClient` for JSON requests.