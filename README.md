# gRPC-Web: Typed Frontend Development


[![Master Build](https://travis-ci.org/improbable-eng/grpc-web.svg)](https://travis-ci.org/improbable-eng/grpc-web)
[![NPM](https://img.shields.io/npm/v/grpc-web-client.svg)](https://www.npmjs.com/package/grpc-web-client) 
[![GoDoc](http://img.shields.io/badge/GoDoc-Reference-blue.svg)](https://godoc.org/github.com/improbable-eng/grpc-web/go/grpcweb) 
[![Apache 2.0 License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](LICENSE)
[![quality: alpha](https://img.shields.io/badge/quality-alpha-orange.svg)](#status)

[gRPC](http://www.grpc.io/) is a modern, [HTTP2](https://hpbn.co/http2/)-based protocol, that provides RPC semantics using the strongly-typed *binary* data format of [protocol buffers](https://developers.google.com/protocol-buffers/docs/overview) across multiple languages (C++, C#, Golang, Java, Python, NodeJS, ObjectiveC, etc. [gRPC-Web](https://github.com/grpc/grpc/blob/master/doc/PROTOCOL-WEB.md) is a cutting-edge spec that enables invoking gRPC services from *modern* browsers.

Components of the stack are based on Golang and TypeScript:
 * [`grpcweb`](go/grpcweb) - a Go package that wraps an existing `grpc.Server` as a gRPC-Web `http.Handler` for both HTTP2 and HTTP/1.1
 * [`grpcwebproxy`](go/grpcwebproxy) - a Go-based stand-alone reverse proxy for classic gRPC servers (e.g. in Java or C++) that exposes their services over gRPC-Web to modern browsers
 * [`ts-protoc-gen`](https://github.com/improbable-eng/ts-protoc-gen) - a TypeScript plugin for the protcol buffers compiler that provides strongly typed message classes and method definitions
 * [`grpc-web-client`](ts) - a TypeScript gRPC-Web client library for browsers, not meant to be used directly by users

 
## Why?

With gRPC-Web, it is extremely easy to build well defined, easy to reason about APIs between browser frontend code and microservices. Frontend development changes significantly:
 * no more hunting down API documentation - `.proto` is the canonical format for API contracts
 * no more hand-crafted JSON call objects - all requests and responses are strongly typed and code-generated, with hints available in the IDE
 * no more dealing with methods, headers, body and low level networking - everything is handled by `grpc-invoke`
 * no more second-guessing the meaning of error codes - [gRPC status codes](https://godoc.org/google.golang.org/grpc/codes) are a canonical way of representing issues in APIs
 * no more one-off server-side request handlers to avoid concurrent connections - gRPC-Web is based on HTTP2, with multiplexes multiple streams over the [same connection](https://hpbn.co/http2/#streams-messages-and-frames)
 * no more problems streaming data from a server -  gRPC-Web supports both *1:1* RPCs and *1:many* streaming requests
 * no more data parse errors when rolling out new binaries - backwards and forwards-[compatibility](https://developers.google.com/protocol-buffers/docs/gotutorial#extending-a-protocol-buffer) of requests and responses

In short, gRPC-Web moves the interaction between frontend code and microservices from the sphere of hand-crafted HTTP requests to well-defined user-logic methods. 

## Example 

For a self-contained demo of a Golang gRPC service called from a TypeScript projects, see [example](example). It contains most of the initialization code that performs the magic. Here's the application code extracted from the example:

You use `.proto` files to define your service. In this example, one normal RPC (`GetBook`) and one server-streaming RPC (`QueryBooks`):

```proto
syntax = "proto3";

message Book {
  int64 isbn = 1;
  string title = 2;
  string author = 3;
}

message GetBookRequest {
  int64 isbn = 1;
}

message QueryBooksRequest {
  string author_prefix = 1;
}

service BookService {
  rpc GetBook(GetBookRequest) returns (Book) {}
  rpc QueryBooks(QueryBooksRequest) returns (stream Book) {}
}
```

And implement it in Go (or any other gRPC-supported language):

```go
import pb_library "../_proto/examplecom/library"

type bookService struct{
  books []*pb_library.Book
}

func (s *bookService) GetBook(ctx context.Context, bookQuery *pb_library.GetBookRequest) (*pb_library.Book, error) {
	for _, book := range s.books {
		if book.Isbn == bookQuery.Isbn {
			return book, nil
		}
	}
	return nil, grpc.Errorf(codes.NotFound, "Book could not be found")
}

func (s *bookService) QueryBooks(bookQuery *pb_library.QueryBooksRequest, stream pb_library.BookService_QueryBooksServer) error {
	for _, book := range s.books {
		if strings.HasPrefix(s.book.Author, bookQuery.AuthorPrefix) {
			stream.Send(book)
		}
	}
	return nil
}
```

You will be able to access it in a browser using TypeScript (and equally JavaScript after transpiling):

```ts
import {grpc, BrowserHeaders} from "grpc-web-client";
// Import code-generated deta structures.
import {BookService} from "./services";
import {QueryBooksRequest, Book, GetBookRequest} from "../_proto/examplecom/library/book_service_pb";

const queryBooksRequest = new QueryBooksRequest();
queryBooksRequest.setAuthorPrefix("Geor");
grpc.invoke(BookService.QueryBooks, {
  request: queryBooksRequest,
  host: host,
  onMessage: function(message: Book) {
    console.log("got book: ", message.toObject());
  },
  onComplete: function(code: grpc.Code, msg: string | undefined, trailers: BrowserHeaders) {
    if code == grpc.Code.OK {
      console.log("all ok")
    } else {
      console.log("hit an error", code, msg, trailers);
    }
  }
});

```

## Browser Support

The `grpc-web-client` uses multiple techniques to efficiently invoke gRPC services. Most [modern browsers](http://caniuse.com/#feat=fetch) support the [Fetch API](https://developer.mozilla.org/en/docs/Web/API/Fetch_API), which allows for efficient reading of partial, binary responses. For older browsers, it automatically falls back to [`XMLHttpRequest`](https://developer.mozilla.org/nl/docs/Web/API/XMLHttpRequest).

The gRPC semantics encourage you to make multiple requests at once. With most modern browsers [supporting HTTP2](http://caniuse.com/#feat=http2), these can be executed over a single TLS connection. For older browsers, gRPC-Web falls back to HTTP/1.1 chunk responses.

For best results we recommend the browsers we test against:
  * Chrome >= 42
  * Firefox >= 39
  * Edge >= 14 
  * Safari >= 10 (with 10.1 significantly improving matters)

### Client-side streaming

It is very important to note that the gRPC-Web spec currently *does not support client-side streaming*. This is unlikely to change until until new whatwg fetch/[streams API](https://www.w3.org/TR/streams-api/) lands in browsers. As such, if you plan on using gRPC-Web you're limited to:
 * unary RPCs (1 request 1 response)
 * server-side streaming RPCs (1 request N responses)

This, however is useful for a lot of frontend functionality.

## Status

The code here is `alpha` quality. It is being used for a subset of Improbable's frontend single-page apps in production.

### Running the tests

Install the `localhost` certificates of this repo found in `misc/`. Follow [this guide](http://stackoverflow.com/questions/7580508/getting-chrome-to-accept-self-signed-localhost-certificate) for Chrome.

Run the TypeScript tests against the Golang TestServer
```
cd  ${GOPATH}/github.com/improbable-eng/grpc-web/test
npm test
```
Point your browser at https://localhost:9876



