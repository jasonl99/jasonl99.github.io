---
layout: post
title: Sockets Are Awesome Part 2
---
In [Sockets Are Awesome: Part 1]({{ site.baseurl }}{% post_url 2017-1-20-managing_sockets_part_1 %})
we had four goals and accomplished two of them:

* ~~All pages have a session (via [kemal-session](https://github.com/kemalcr/kemal-session))~~
* ~~All pages immediately, before any other javascript, create a socket connection back to the server~~
* All sockets must be associated with a session
* All data communicated over the socket is valid JSON.

At this stage, we have a web server running with a big list of sessions for each user that we can access
using `Session`. And we also have one or more websocket connection for each of those users.  But
we _don't_ have them to associated with each other.  So that's the goal - when a message comes in over
a web socket, we want to know which user (via their session) sent the message.  And vice-versa:  if
have somwething on the server that we want to send to the user (unread message badge, etc) we
want to know which session to send it across.

So we have to connect a socket, initiated by the browser, to a session, which was initiated by the
web server.  At first, I thought this would be easy - just use javascript to find the cookie
for the session and use that for a url.  Well, there are few odditites of doing that, for example
`localhost` for some reason [doesn't work](http://stackoverflow.com/questions/1134290/cookies-on-localhost-with-explicit-domain)
with chrome when reading cookies.  We could work around that, but it also just seems messy -
we'd have to know the name of the cookie that holds the session identifier (which is conigured
with `Session.config`) and that just reeks of stinky code.

So then I started thinking maybe there's a way to capture the session_id somwhere in the chain
of handlers that crystal uses to process an http request.  And adding HTTP::Handler and dumping
the output showed that yes, a request to a websocket generates a regular HTTP request, complete
with a kemal session cookie.  But somewhere along its handler journey, it gets converted to 
a WebSocketHandler, and the kemal session cookie is no longer available.  

Consider these two kemal routing handlers, one for a page, and one for a socket:

```ruby
get "/page1" do |context|
  #context has the following properties"
  #  request = information about the request to /page1 including headers, cookies, etc.
  #  session = a kemal-session instance
  #  response - the server's response to the request
  #
  # render HTML here
end

ws "/socket1" do |socket|
 # we don't send anything.  We create additional handlers:
 # socket has no useful properties for us.  It has some internal 
 # IO stuff, but no request, context, or response properties.  So 
 # we there's no way in this method to find which browser created
 # the socket (and, consequently, which session it came from).
 socket.on_message do |msg|
    puts "The socket sent a message to the server"
  end
end
```

It seems like `socket` parameters passed into the `#ws` block could have this info, since
it did exist at some point earlier in the handler chain.  But it doesn't, so there's two options
at this point:  1) monkey-patch kemal or crystal's http, or 2) find another way.

It then occurred to me that there's a _very_ simple way to do this, though it does mess with
encapsulation a bit.  Since we have the session available in the first route, `get "page1"`, 
and we already have a little chunk of javascript that creates the socket with 
`new WebSocket("ws:" + location.host "/socket1")`, we can leverage this a with just a bit more code.

Javascript's implementation of websocket (which we'll use in more depth later), has a convenient
method `#onconnect` which is called the first time a connection is established and ready.  This 
is perfect!  We can send the session_id right back to the server from the browser.  Here's what the code looks
like to send a JSON object with a key "sessionID":

```ruby
get "/page1" do |context|
  javascript = <<-JS
    app_vars = { ws: new WebSocket("ws:" + location.host + "/chat")};
    app_vars.ws.onmessage = function(evt) { console.log(evt.data) };
    app_vars.ws.onopen = function(evt) {                         # this is our
    evt.target.send(                                             # way to send back
      JSON.stringify( {sessionID: "#{context.session.id}"})      # the sessionID.
      );
    }
  JS
  render "page.slang"

end
```
The downside to this approach is that every page now needs that little bit of javascript to tie
socket back to the session.

### Back To The Server

Ok, so now we have the client creating a connection and immediately sending back to the server 
a means to directly identify the owner of this socket.  So we have to process it.

Back at the server, we add an `on_message` handler to the socket. 

```ruby
ws "/socket1" do |socket|
  socket.on_message do |message|
    # message is a simple string that needs to be parsed into JSON
    payload = JSON.parse message
    session_id = payload["sessionID"].as_s? if payload["sessionID"]?
  end
end

```

So we now have a socket and a session_id.  So how do we associated them?  We have to create
an association class whose instances contain a socket and a session.  The majority of the action
that occurs in this instance will be reacting to or sending socket messages.   Since sessions
don't have events, but sockets do, this makes sense.  So let's create such a class, and for
what it's worth, call it SocketManager, since it manages the socket's association with our user.

We use `Session.get` to find the actual session instance for the session_id and set it in the
SocketManager instance.

```ruby
class SocketManager
  socket   : HTTP::Socket
  session  : Session?

  # we can't create a SocketManager instnace without a socket, but we can
  # have one without a session while the socket is sending back its session_id.
  def initialize(@socket)
    socket.on_message {|msg| self.on_message msg}
  end

  # every message that comes across the socket will end up here.
  # We're expecting our sessionID to come through as a JSON message
  def on_message(message)
    payload = JSON.parse message
    if payload["sessionID"]? && (session_id = payload["sessionID"].as_s?)
      self.session = Session.get session_id  # returns a reference to the instantiated session
    end
  end

  # we could just use #delegate but for now let's have more control
  def send(message)
    socket.send(message)
  end

end
```

We still have to create a SocketManager instance, but that's easy now. In our kemal route, we just
do this:

```ruby
ws "/socket1" do |socket|
  SocketManager.new(socket)
end
```

That's enough for now.  It'll get tied up in Part 3 with a comprehensive SocketManager class that
even manages routing.
