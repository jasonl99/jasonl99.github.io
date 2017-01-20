---
layout: post
title: Sockets Are Awesome
---
Crystal's standard library contains a phenomenal class that has the opportunity to change the
way you develop websites, and I mean radically change it.

In fact, my inspiration for trying my hand at writing a framework is, in large part, due to the
power it provides.  It's `HTTP::WebSocket`.  Up until a few weeks ago, I had barely given 
[websockets](https://www.websocket.org/quantum.html) a glance.  It seemed like something people 
used for video or voip calls or stuff I wasn't all that interested in.

But then I started digging into it.  Crystal (and kemal, in particular) made this easy.  And after 
a few weeks, I can say that websockets are
__FREAKING AMAZING__.  It is sooo much better than AJAX, or polling, or any other crazy
method (*cough* hack *cough*) you've used to synchronize data between the web server and browser.

It's part of the HTML5 spec, and it's supported on the _current_ version of
_all_ browsers.  Is that a limitation for where it can be used? Well, yes, but only if you
intend to support older browsers.  I'm tired of that world.  If someone doesn't want to update
their browser, that's their choice.  But this framework is going to be designed, from the ground up, to 
leverage websockets  __extensively__. It is going to __assume__ that the browser supports them.
Sorry.  I'd rather not get into a philosophical debate about it, and I won't be offended it you
think a websockets-first framework is a dumb idea (and, just so we're clear, it's entirely
possible that it is a dumb idea, and just don't see it).

So, with that admittedly rude intro, let's plan to build a framework:

[kemal](kemalcr.com) is a great base to start with.  It handles all the web routing verbs, has a session manager
that frees us from having to roll our own security, and is basically __exactly__ the foundation 
needed.  From this base, we want to accomplish the following goals:

* All pages have a session (via [kemal-session](https://github.com/kemalcr/kemal-session))
* All pages immediately, before any other javascript, create a socket connection back to the server
* All sockets must be associated with a session
* All data communicated over the socket is valid JSON.

This won't do anything for displaying data, but it will create a reliable and efficient full
duplex channel between the server and brower, and it's by its nature asynchronous.

We need shards:   `/shard.yml`

```yaml
dependencies:
  kemal:
    github: kemalcr/kemal
    branch: master
  kemal-session:
    github: kemalcr/kemal-session
    branch: master
  slang:
    github: jeromegn/slang

crystal: 0.20.3

license: MIT
```

I've already added slang as a template rendering engine, because, well, it's awesome.

I'm not sure where the code will end up yet, but a crystal lib or app defines a module 
when you init an app.  So we'll start there, and call the framework "Lattice"


```ruby
require "./lattice/*"
require "kemal"
require "kemal-session"
require "kilt/slang"

module Lattice
  get "/*" do |context|
    context.session.string("phrase",sample_phrase) unless context.session.string?("phrase")
    render "page.slang"
  end
  def sample_phrase
    %w(sunshine rainbows ruby crystal chair mazda puppy coat").sample(2).join("-")
  end
end
```

Now we need to render a page.  In `page.slang`:

```slim
head
body
  h1 Hi.
  = context.session.string?("phrase")
```

All this does is wire up some of the parts we're using, and verifies it on page load.  It also
lets us test for persisted values in a session (so we can test if the socket is connected
correctly to the session).

So at this point running the app with `crystal src/lattice.cr` will let you go to 
any url on localhost, for example, `http://localhost:3000/test`, and you'll see a phrase.

We next have to create a websockets  from the session back to the server.  This __must__ be
done with javascript, so we'll need to serve it up.  For now, we'll render it dynamically
because we'll need to shortly anyway to tie it to a session.

So there's a few changes to `./src/lattice.cr`

```ruby
  get "/*" do |context|
    context.session.string("phrase",sample_phrase) unless context.session.string?("phrase")
    javascript = <<-JS
      app_vars = { ws: new WebSocket("ws:" + location.host + "/create_socket")};
    JS
    render "page.slang"
  end
```

I am not a javascript expert.  But this websocket must persist, and I want it namespaced, so
I create a js object (app_vars) and create a websocket inside it.  I don't know if this is
the ideal way to do it, but it seems reasonable to me.  This just creates a string
of javascript code that will be rendered on `page.slang`

```slim
head
  script
    == javascript
body
  h1 Hi.
  = context.session.string?("phrase")
```

If we run the app now, we get no change, but if we open the console on the browser we get this:
```
VM3782:1 WebSocket connection to 'ws://localhost:3000/create_socket' failed: Error during WebSocket 
```

This is happening because we haven't created a route to `/create_socket`.  Which is easy enough,
just add it to our module where we already have a `#get`.  We add the object_id to make 
it easy to see different sockets (you can load from firefox and chrome separately, for example).

```ruby
ws "/create_socket" do |socket|
  puts "Socket created #{socket.object_id}"
end
```

Reload again, and viola.  We now have a socket connected.  No error on the client.  In fact,
on the console, `app_vars.ws.readyState` should be 1 which means the socket is open and ready
to send and receive data.

But on the server, once we finish the `#ws` method, the socket disappears from our knowledge.
We have no way to react to incoming data, and no way to send outgoing data.  we've lost any
reference to the socket that was created.  But we can, inside the context of ws, create a 
block that reacts to incoming data.  It logs it to the terminal, and then thanks the browser
for sending it.

```ruby
  ws "/create_socket" do |socket|
    puts "Socket created #{socket.object_id}"
    socket.on_message do |message|
      puts "Just received a message from #{socket.object_id}: #{message}"
      socket.send "Thanks for sending the message '#{message}'"
    end
  end
```

Run the app again, reload the page, and go to the console. Let's send data back
to the server:

```javascript
app_vars.ws.send("Hello from the browser")
```

In the terminal, where are app is runnig, you should see
```
Just received a message from 94147949214224: Hello from the browser
```

We almost have the entire cycle of communication established.  We still have to 
react to _incoming_ data on the browser.  Sockets objects in javascript on the browser 
have an `#onmessage` event that we can use.  So we need to add this to our
app_vars.ws object in `./src/lattice.cr`:

```ruby
  javascript = <<-JS
    app_vars = { ws: new WebSocket("ws:" + location.host + "/chat")};
    app_vars.ws.onmessage = function(evt) { console.log(evt.data) };
  JS
```

this will simply echo any data from the server back to the console.  So start the app, and
try again:

```
Thanks for sending the message 'Hello from the browser'
```

So that's in for Part 1.  At the outset, we had four goals, and we've accomplished two of
them.  In [Part 2](managing_sockets_part_2.md), we'll connect our sockets to a session.


