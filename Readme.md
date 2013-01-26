# Socket.IO

Socket.IO is a Node.JS project that makes WebSockets and realtime possible in
all browsers. It also enhances WebSockets by providing built-in multiplexing,
horizontal scalability, automatic JSON encoding/decoding, and more.

Socket.IO是一个让所有浏览器支持WebSockets和实时通信成为可能的Node.JS项目。这也增强了通过提供内置多线程(实现)的WebSockets，横向可扩展性，自动JSON编码/解码，等等。

## How to Install（如何安装）

```bash
npm install socket.io
```

## How to use（如何使用）

First, require `socket.io`:

首先，require `socket.io`：

```js
var io = require('socket.io');
```

Next, attach it to a HTTP/HTTPS server. If you're using the fantastic `express`
web framework:

然后，把它连接到一个http/https服务器。如果你正使用神奇的`express`web框架：

#### Express 3.x

```js
var app = express()
  , server = require('http').createServer(app)
  , io = io.listen(server);

server.listen(80);

io.sockets.on('connection', function (socket) {
  socket.emit('news', { hello: 'world' });
  socket.on('my other event', function (data) {
    console.log(data);
  });
});
```

#### Express 2.x

```js
var app = express.createServer()
  , io = io.listen(app);

app.listen(80);

io.sockets.on('connection', function (socket) {
  socket.emit('news', { hello: 'world' });
  socket.on('my other event', function (data) {
    console.log(data);
  });
});
```

Finally, load it from the client side code:

最后，在客户端代码中加载它：

```html
<script src="/socket.io/socket.io.js"></script>
<script>
  var socket = io.connect('http://localhost');
  socket.on('news', function (data) {
    console.log(data);
    socket.emit('my other event', { my: 'data' });
  });
</script>
```

For more thorough examples, look at the `examples/` directory.

对于更深入的案例，查看`examples/`目录。

## Short recipes（快捷方法）

### Sending and receiving events.（发送和接收事件）

Socket.IO allows you to emit and receive custom events.

Sicket.IO允许你emit和receive自定义事件。

Besides `connect`, `message` and `disconnect`, you can emit custom events:

除了`connect`，`message`和`disconnect`，你还可以emit自定义事件：

```js
// note, io.listen(<port>) will create a http server for you
var io = require('socket.io').listen(80);

io.sockets.on('connection', function (socket) {
  io.sockets.emit('this', { will: 'be received by everyone' });

  socket.on('private message', function (from, msg) {
    console.log('I received a private message by ', from, ' saying ', msg);
  });

  socket.on('disconnect', function () {
    io.sockets.emit('user disconnected');
  });
});
```

### Storing data associated to a client（存储与客户端关联的数据）

Sometimes it's necessary to store data associated with a client that's
necessary for the duration of the session.

有时候，需要依赖持续的会话存储与客户端相关联的数据。

#### Server side（服务端）

```js
var io = require('socket.io').listen(80);

io.sockets.on('connection', function (socket) {
  socket.on('set nickname', function (name) {
    socket.set('nickname', name, function () { socket.emit('ready'); });
  });

  socket.on('msg', function () {
    socket.get('nickname', function (err, name) {
      console.log('Chat message by ', name);
    });
  });
});
```

#### Client side（客户端）

```html
<script>
  var socket = io.connect('http://localhost');

  socket.on('connect', function () {
    socket.emit('set nickname', prompt('What is your nickname?'));
    socket.on('ready', function () {
      console.log('Connected !');
      socket.emit('msg', prompt('What is your message?'));
    });
  });
</script>
```

### Restricting yourself to a namespace（限制自己的命名空间）

If you have control over all the messages and events emitted for a particular
application, using the default `/` namespace works.

如果你已经控制到特定应用的所有消息和事件，使用默认的`/`命名空间工作。

If you want to leverage 3rd-party code, or produce code to share with others,
socket.io provides a way of namespacing a `socket`.

如果你想使用第三方的代码，或者将生产的代码分享给其他人，socket.io提供了一个方法来命名一个`socket`。

This has the benefit of `multiplexing` a single connection. Instead of
socket.io using two `WebSocket` connections, it'll use one.

`multiplexing`一个单链接是有优势的。它使用一个链接，而不是socket.io使用两次`WebSocket`链接。

The following example defines a socket that listens on '/chat' and one for
'/news':

下面的案例定义了一个socket来监听'/chat'和'/news'：

#### Server side（服务端）

```js
var io = require('socket.io').listen(80);

var chat = io
  .of('/chat')
  .on('connection', function (socket) {
    socket.emit('a message', { that: 'only', '/chat': 'will get' });
    chat.emit('a message', { everyone: 'in', '/chat': 'will get' });
  });

var news = io
  .of('/news');
  .on('connection', function (socket) {
    socket.emit('item', { news: 'item' });
  });
```

#### Client side:（客户端：）

```html
<script>
  var chat = io.connect('http://localhost/chat')
    , news = io.connect('http://localhost/news');

  chat.on('connect', function () {
    chat.emit('hi!');
  });

  news.on('news', function () {
    news.emit('woot');
  });
</script>
```

### Sending volatile messages.（发送易变的消息。）

Sometimes certain messages can be dropped. Let's say you have an app that
shows realtime tweets for the keyword `bieber`. 

有时候，某些信息能被取消。比方说你有一个显示`bieber`关键词实时鸣叫的应用。

If a certain client is not ready to receive messages (because of network slowness
or other issues, or because he's connected through long polling and is in the
middle of a request-response cycle), if he doesn't receive ALL the tweets related
to bieber your application won't suffer.

如果某一个客户端还没有准备好接受消息（因为网络慢或其他问题，或者因为他已经连接上一个长轮询，并且在一个“请求”-“响应”周期中），如果他不接收所有的bieber鸣叫你的应用不会受到影响。

In that case, you might want to send those messages as volatile messages.

（但是）在这种情形下，你可能想要发送这些易失的信息。

#### Server side

```js
var io = require('socket.io').listen(80);

io.sockets.on('connection', function (socket) {
  var tweets = setInterval(function () {
    getBieberTweet(function (tweet) {
      socket.volatile.emit('bieber tweet', tweet);
    });
  }, 100);

  socket.on('disconnect', function () {
    clearInterval(tweets);
  });
});
```

#### Client side

In the client side, messages are received the same way whether they're volatile
or not.

在客户端，消息是否丢失的接收方式相同。

### Getting acknowledgements（获得确认）

Sometimes, you might want to get a callback when the client confirmed the message
reception.

有时候，当客户端确认接收到消息后你可能想要得到回馈。

To do this, simply pass a function as the last parameter of `.send` or `.emit`.
What's more, when you use `.emit`, the acknowledgement is done by you, which
means you can also pass data along:

要做到这一点，简单的传递一个函数作为`.send`或`.emit`的最后一个参数。更重要的是，当你使用`.emit`的时候，确认是你自己做的，那就意味着你也可以只传递数据：

#### Server side

```js
var io = require('socket.io').listen(80);

io.sockets.on('connection', function (socket) {
  socket.on('ferret', function (name, fn) {
    fn('woot');
  });
});
```

#### Client side

```html
<script>
  var socket = io.connect(); // TIP: .connect with no args does auto-discovery
  socket.on('connect', function () { // TIP: you can avoid listening on `connect` and listen on events directly too!
    socket.emit('ferret', 'tobi', function (data) {
      console.log(data); // data will be 'woot'
    });
  });
</script>
```

### Broadcasting messages（广播消息）

To broadcast, simply add a `broadcast` flag to `emit` and `send` method calls.
Broadcasting means sending a message to everyone else except for the socket
that starts it.

广播，只需要在`.emit`和`.send`方法调用的时候添加一个`broadcast`标记。广播是指发送消息到除本人以外的所有的时候使用它。

#### Server side

```js
var io = require('socket.io').listen(80);

io.sockets.on('connection', function (socket) {
  socket.broadcast.emit('user connected');
  socket.broadcast.json.send({ a: 'message' });
});
```

### Rooms（空间）

Sometimes you want to put certain sockets in the same room, so that it's easy
to broadcast to all of them together.

有时候你想要把某些sockets放到相同的空间，这样就可以简单的广播到在一起的所有人。

Think of this as built-in channels for sockets. Sockets `join` and `leave`
rooms in each socket.

想想这是为sockets提供的内置通道。每一个socket`join`和`leave`sockets空间。

#### Server side

```js
var io = require('socket.io').listen(80);

io.sockets.on('connection', function (socket) {
  socket.join('justin bieber fans');
  socket.broadcast.to('justin bieber fans').emit('new fan');
  io.sockets.in('rammstein fans').emit('new non-fan');
});
```

### Using it just as a cross-browser WebSocket（只是作为一个跨浏览器的WebSocket使用它）

If you just want the WebSocket semantics, you can do that too.
Simply leverage `send` and listen on the `message` event:

如果你只是想要WebSocket语义，你也可以做到这点。只需使用`send`和监听`message`事件：

#### Server side

```js
var io = require('socket.io').listen(80);

io.sockets.on('connection', function (socket) {
  socket.on('message', function () { });
  socket.on('disconnect', function () { });
});
```

#### Client side

```html
<script>
  var socket = io.connect('http://localhost/');
  socket.on('connect', function () {
    socket.send('hi');

    socket.on('message', function (msg) {
      // my msg
    });
  });
</script>
```

### Changing configuration（更改配置）

Configuration in socket.io is TJ-style:

socket.io中的配置是TJ-style：

#### Server side

```js
var io = require('socket.io').listen(80);

io.configure(function () {
  io.set('transports', ['websocket', 'flashsocket', 'xhr-polling']);
});

io.configure('development', function () {
  io.set('transports', ['websocket', 'xhr-polling']);
  io.enable('log');
});
```

## License 

(The MIT License)

Copyright (c) 2011 Guillermo Rauch &lt;guillermo@learnboost.com&gt;

Permission is hereby granted, free of charge, to any person obtaining
a copy of this software and associated documentation files (the
'Software'), to deal in the Software without restriction, including
without limitation the rights to use, copy, modify, merge, publish,
distribute, sublicense, and/or sell copies of the Software, and to
permit persons to whom the Software is furnished to do so, subject to
the following conditions:

The above copyright notice and this permission notice shall be
included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED 'AS IS', WITHOUT WARRANTY OF ANY KIND,
EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF
MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT.
IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY
CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT,
TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE
SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.
