# Golang Chatrooms Example: Independent Rooms

*This project adapted from [the gorilla websocket chat example](https://github.com/gorilla/websocket/tree/master/examples/chat) to support multiple chat rooms and JSON websocket message data*

This application shows how to use the
[websocket](https://github.com/gorilla/websocket) package to implement a simple
web chat application with multiple rooms

## Running the example

The example requires a working Go development environment. The [Getting
Started](http://golang.org/doc/install) page describes how to install the
development environment.

Once you have Go up and running, you can download, build and run the example
using the following commands.

    $ go get github.com/gorilla/websocket
    $ git clone https://github.com/ibraheemdev/go-chatrooms-example
    $ cd go-chatrooms-example
    $ go run .

To use the chat example, open http://localhost:8080/ in your browser.

## Server

The server application defines two types, `Client` and `Room`. The server
creates an instance of the `Client` type for each websocket connection. A
`Client` acts as an intermediary between the websocket connection and a `Room` instance. 
A `Room` maintains a set of registered clients and
broadcasts messages to the clients.

The application runs one goroutine for each `Room` and two goroutines for each
`Client`. The goroutines communicate with each other using channels. A `Room`
has channels for registering clients, unregistering clients and broadcasting
messages. A `Client` has a buffered channel of outbound messages. One of the
client's goroutines reads messages from this channel and writes the messages to
the websocket. The other client goroutine reads messages from the websocket and
sends them to the room.

### Room 

The code for the `Room` type is in
[room.go](https://github.com/ibraheemdev/go-chat/blob/master/room.go). 
The application's `main` function starts each room's`run` method as a goroutine.
Clients send requests to the room using the `register`, `unregister` and
`broadcast` channels.

The room registers clients by adding the client pointer as a key in the
`clients` map. The map value is always true.

The unregister code is a little more complicated. In addition to deleting the
client pointer from the `clients` map, the room closes the clients's `send`
channel to signal the client that no more messages will be sent to the client.

The room handles messages by looping over the registered clients and sending the
message to the client's `send` channel. If the client's `send` buffer is full,
then the room assumes that the client is dead or stuck. In this case, the room
unregisters the client and closes the websocket.

### Client

The code for the `Client` type is in [client.go](https://github.com/gorilla/websocket/blob/master/examples/chat/client.go).

The `serveWs` function is registered by the application's `main` function as
an HTTP handler. The handler upgrades the HTTP connection to the WebSocket
protocol, creates a client, registers the client with the room and schedules the
client to be unregistered using a defer statement.

Next, the HTTP handler starts the client's `writePump` method as a goroutine.
This method transfers messages from the client's send channel to the websocket
connection. The writer method exits when the channel is closed by the room or
there's an error writing to the websocket connection.

Finally, the HTTP handler calls the client's `readPump` method. This method
reads the messages into a Message struct. This is when database calls or 
other logic should be handled. It then marshals it and transfers inbound messages 
from the websocket to the room.

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
