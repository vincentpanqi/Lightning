<p align="center">
<img src="https://user-images.githubusercontent.com/6432361/34271845-06608678-e65c-11e7-896a-a4eeb2991300.png" height="224" alt="Edge">
<br/>Serverside non-blocking IO <b>in Swift</b><br/>
Ask questions in our <a href="https://slackin-on-edge.herokuapp.com">Slack</a> channel!<br/>
</p>


# Lightning
##### (formerly Edge)

![Swift](http://img.shields.io/badge/swift-4.0.2-brightgreen.svg)
[![Build Status](https://travis-ci.org/skylab-inc/Lightning.svg?branch=master)](https://travis-ci.org/skylab-inc/Lightning)
[![codecov](https://codecov.io/gh/skylab-inc/Lightning/branch/master/graph/badge.svg)](https://codecov.io/gh/skylab-inc/Lightning)
[![Slack Status](https://slackin-on-edge.herokuapp.com/badge.svg)](https://slackin-on-edge.herokuapp.com)

#### Node
Lightning is an HTTP Server and TCP Client/Server framework written in Swift and inspired by [Node.js](https://nodejs.org). It runs on both OS X and Linux. Like Node.js, Lightning uses an **event-driven, non-blocking I/O model**. In the same way that Node.js uses [libuv](http://libuv.org) to implement this model, Lightning uses [libdispatch](https://github.com/apple/swift-corelibs-libdispatch). 

This makes Lightning fast, efficient, and most crutially **single-threaded** by default. You simply do not need to worry about locks/mutexes/semaphores/etc if you have server-side state. Of course, Lightning applications can make use of libdispatch to easily offload heavy processing to a background thread if necessary.

#### Reactive Programming
Lightning's event API embraces Functional Reactive Programming by generalizing the familiar concept of promises. This API is called [StreamKit](https://github.com/skylab-inc/StreamKit).

> StreamKit's architecture is inspired by both [ReactiveCocoa](https://github.com/ReactiveCocoa/ReactiveCocoa) and [RxSwift](https://github.com/ReactiveX/RxSwift).

##### Why did we reimplement?
* Lightning should be easy to use out of the box.
* Lightning is optimized for maximum performance, which requires careful tuning of the internals.
* The modified API is meant to be more similar to the familiar concepts of Futures and Promises.
* We don't want to be opinionated about any one framework. We want it to be easy to integate Lightning with either ReactiveCocoa or RxSwift.

>FRP, greatly simplies management of asynchronous events. The general concept is that we can build a spout which pushes out asynchronous events as they happen. Then we hookup a pipeline of transformations that operate on events and pass the transformed values along. We can even do things like merge streams in interesting ways! Take a look at some of these [operations](http://rxmarbles.com) or watch [this talk](https://www.youtube.com/watch?v=XRYN2xt11Ek) about how FRP is used at Netflix. 

# Installation

Lightning is available as a Swift 3/4 package. Simply add Lightning as a dependency to your Swift Package.

Swift 3
```Swift
import PackageDescription

let package = Package(
    name: "MyProject",
    dependencies: [
        .Package(url: "https://github.com/skylab-inc/Lightning.git", majorVersion: 0, minor: 3)
    ]
)
```
Swift 4
```Swift
// swift-tools-version:4.0
// The swift-tools-version declares the minimum version of Swift required to build this package.
import PackageDescription

let package = Package(
    name: "MyProject",
    dependencies: [
        .package(url: "https://github.com/skylab-inc/Lightning.git", from: "0.3.0"),
    ]
)
```

# Usage

### Routing
```swift
import Lightning
import Foundation

// Create an API router.
let api = Router()

// Add a GET "/users" endpoint.
api.get("/users") { request in
    return Response(status: .ok)
}

// NOTE: Equivalent to `api.post("/auth/login")`
let auth = api.subrouter("/auth")
auth.post("/login") { request in
    return Response(status: .ok)
}

// Middleware to log all requests
// NOTE: Middleware is a simple as a map function or closure!
let app = Router()
app.map { request in
    print(request)
    return request
}

// Mount the API router under "/v1.0".
app.add("/v1.0", api)

// NOTE: Warnings on all unhandled requests. No more hanging clients!
app.any { _ in
    return Response(status: .notFound)
}

// Start the application.
app.start(host: "0.0.0.0", port: 3000)
```

### Raw HTTP
```swift
import Lightning
import Foundation

func handleRequest(request: Request) -> Response {
    print(String(bytes: request.body, encoding: .utf8)!)
    return try! Response(json: ["message": "Message received!"])
}

let server = HTTP.Server()
server.listen(host: "0.0.0.0", port: 3000).startWithNext { client in

    let requestStream = client.read()
    requestStream.map(handleRequest).onNext{ response in
        client.write(response).start()
    }

    requestStream.onFailed { clientError in
        print("Oh no, there was an error! \(clientError)")
    }

    requestStream.onCompleted {
        print("Goodbye \(client)!")
    }

    requestStream.start()
}

RunLoop.runAll()
```

### TCP
```Swift

import Lightning
import Foundation

let server = try! TCP.Server()
try! server.bind(host: "0.0.0.0", port: 50000)
    
server.listen().startWithNext { connection in
    let byteStream = connection.read()
    let strings = byteStream.map { String(bytes: $0, encoding: .utf8)! }
    
    strings.onNext { message in
        print("Client \(connection) says \"\(message)\"!")
    }
    
    strings.onFailed { error in
        print("Oh no, there was an error! \(error)")
    }
    
    strings.onCompleted {
        print("Goodbye \(connection)!")
    }
    
    strings.start()
}

RunLoop.runAll()
```


### Lightning is not Node.js

Lightning is not meant to fulfill all of the roles of Node.js. Node.js is a JavaScript runtime, while Lightning is a TCP/Web server framework. The Swift compiler and package manager, combined with third-party Swift packages, make it unnecessary to build that functionality into Lightning.
