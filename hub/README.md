# Golang Chatrooms Example: Hub

*This project adapted from [the gorilla websocket chat example](https://github.com/gorilla/websocket/tree/master/examples/chat) to support multiple chat rooms and JSON websocket message data*

This application shows how to use the
[websocket](https://github.com/gorilla/websocket) package to implement a simple
web chat application.

## Running the example

The example requires a working Go development environment. The [Getting
Started](http://golang.org/doc/install) page describes how to install the
development environment.

Once you have Go up and running, you can download, build and run the example
using the following commands.

    $ go get github.com/gorilla/websocket
    $ cd `go list -f '{{.Dir}}' github.com/gorilla/websocket/examples/chat`
    $ go run *.go

To use the chat example, open http://localhost:8080/ in your browser.

## Server

The server application defines two types, `Client` and `Hub`. The server
creates an instance of the `Client` type for each websocket connection. A
`Client` acts as an intermediary between the websocket connection and a single
instance of the `Room` type. The `Hub` maintains a set of registered rooms and
broadcasts messages to the clients in the room the message was sent to.

The application runs one goroutine for the `Hub` and two goroutines for each
`Client`. The goroutines communicate with each other using channels. The `Hub`
has channels for registering clients, unregistering clients and broadcasting
messages. A `Client` has a buffered channel of outbound messages. One of the
client's goroutines reads messages from this channel and writes the messages to
the websocket. The other client goroutine reads messages from the websocket and
sends them to the hub.

### Hub 

The code for the `Hub` type is in
[hub.go](https://github.com/gorilla/websocket/blob/master/examples/chat/hub.go). 
The application's `main` function starts the hub's `run` method as a goroutine.
Clients send requests to the hub using the `register`, `unregister` and
`broadcast` channels.

When a client registers to the hub, it checks if the room sent by the client exists in the `rooms` map. If it doesn't the hub registers the room by adding the room's name as a key in the map. The room then registers the client by adding the client pointer as a key in the `clients` map. The map value is always true.

The unregister code is a little more complicated. In addition to deleting the
client pointer from the `clients` map, the hub closes the clients's `send`
channel to signal the client that no more messages will be sent to the client. If the room's `client` map is empty, then the room is deleted from the hub's `rooms` map.

The hub handles messages by looping over the registered clients in the room and sending the message to the client's `send` channel. If the client's `send` buffer is full, then the hub assumes that the client is dead or stuck. In this case, the hub unregisters the client and closes the websocket, and closes the room if it is empty.

### Client

The code for the `Client` type is in [client.go](https://github.com/gorilla/websocket/blob/master/examples/chat/client.go).

The `serveWs` function is registered by the application's `main` function as
an HTTP handler. The handler upgrades the HTTP connection to the WebSocket
protocol, creates a client, registers the client with the room and schedules the
client to be unregistered using a defer statement.

Next, the HTTP handler starts the client's `writePump` method as a goroutine.
This method transfers messages from the client's send channel to the websocket
connection. The writer method exits when the channel is closed by the hub or
there's an error writing to the websocket connection.

Finally, the HTTP handler calls the client's `readPump` method. This method decodes messages into a Message struct, and transfers them from the websocket to the hub.

WebSocket connections [support one concurrent reader and one concurrent
writer](https://godoc.org/github.com/gorilla/websocket#hdr-Concurrency). The
application ensures that these concurrency requirements are met by executing
all reads from the `readPump` goroutine and all writes from the `writePump`
goroutine.

To improve efficiency under high load, the `writePump` function coalesces
pending chat messages in the `send` channel to a single WebSocket message. This
reduces the number of system calls and the amount of data sent over the
network.

## Frontend

The frontend code is in home.html.

When the client clicks 'join room', the script checks for websocket functionality in the browser. If websocket functionality is available, then the script opens a connection to the server and registers a callback to handle messages from the server. The callback appends the message to the chat log. Leaving a room closes the websocket connection, and removes all messages from the log.

The form handler writes the user input to the websocket and clears the input field.