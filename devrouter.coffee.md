Devrouter Service
=================

This service routes http connections based on hostname and an internal routing
table. That routing table can be initialised in a config file, but is mostly
designed to be set in real time by running the `devroute` command.

It expects to have an entire TLD pointed to it, eg `localhost` or `dev`.
To do the latter on OS X, write this to `/etc/resolver/dev`:
```
nameserver 127.0.0.1
port 10053
```

Because it needs to listen on port 80, you might want to run this as root.
Alternatively, there are various non-root options available depending on your
platform.

    fs = require 'fs'
    WebSocketServer = require('ws').Server
    dns = require 'native-dns'
    httpProxy = require 'http-proxy'
    http = require 'http'

    ttl = 60
    dnsPort = 10053
    httpPort = process.env.PORT or 80
    host = '0.0.0.0'
    registryfile = '/etc/devrouter.json'


Registry
--------

This is just a simple object mapping names to ports and owners. The owner
concept is necessary so that we can clean up all the routes created by a
client when it disconnects.

    registry = {}

    addRoutes = (id, routes) ->
      for k, v of routes
        route = {port: v, owner: id}
        registry[k] = route
      console.log "registry", registry

    delRoutes = (id) ->
      for k, v of registry
        delete registry[k] if v.owner is id
      console.log "registry", registry

    try
      addRoutes null, JSON.parse fs.readFileSync registryfile


DNS Server
----------

Pretty simple stuff. We just return our host for every query we get.

    dnsserver = dns.createServer()

    dnsserver.on 'request', (req, res) ->
      res.answer.push dns.A
        name: req.question[0].name,
        address: host,
        ttl: ttl
      res.send()

    dnsserver.on 'error', (err, buff, req, res) ->
      console.log err.stack

    dnsserver.serve dnsPort, host


HTTP Router
-----------

This router is where all the http traffic goes before it hits the devrouted
service. For the root domain (ie 'dev.' or 'localhost' or '127.0.0.1') we
handle the request ourselves with a homepage. For everything else we look the
port up in our registry and route the traffic there.


Our homepage is just a list of routed names/ports.

    home = (req, res) ->
      return res.end ("#{name}.dev => localhost:#{port}" for name, {port} of registry).join '\n'

This is a helper to turn a request header into a name we can use in our lookup
table. Note that the empty string represents home.

    getName = (req) ->
      return '' if !req.headers.host or req.headers.host.match /^[\d.]+$/
      hostParts = req.headers.host.split '.'
      hostParts.pop() if hostParts[hostParts.length-1] is ''
      return '' unless hostParts.length > 1
      name = hostParts[hostParts.length-2]
      return name

The router is some simple logic around a node http server. We let http-proxy
handle the heavy lifting.

    proxy = httpProxy.createProxyServer()
    proxy.on 'error', (err) -> console.error "proxy error", err

    router = http.createServer (req, res) ->
      name = getName req
      return home(req, res) if !name

      if port = registry[name]?.port
        console.log "http://#{name}#{req.url} -> http://localhost:#{port}#{req.url}"
        proxy.web req, res, target: "http://localhost:#{port}"
      else
        res.statusCode = 404
        res.end "#{name} not found"

We handle websocket connections in much the same way. We have a control
websocket for `devroute` instances on the home route, so we keep those, but
everything else gets proxied.

    router.on 'upgrade', (req, socket, head) ->
      name = getName req
      if name is ''
        wss.handleUpgrade req, socket, head, (client) -> wss.emit 'connection', client
      else if name and port = registry[name]?.port
        console.log "ws://#{name}#{req.url} -> ws://localhost:#{port}#{req.url}"
        proxy.ws req, socket, head, target: "http://localhost:#{port}"
      else
        socket.write('HTTP/1.1 404 Not Found\r\n' +
          'Upgrade: WebSocket\r\n' +
          'Connection: Upgrade\r\n' +
          '\r\n')
        socket.end()


    router.listen httpPort, host


Websocket Controller
--------------------

We use this to add routes. It attaches to the same HTTP server we use for
routing.

The design is that the routes are kept alive by an active websocket
connection, and removed when that connection terminates. The only exception
are routes added within the config file at startup.

    nextid = 0
    wss = new WebSocketServer noServer: true
    wss.on 'connection', (ws) ->
      id = nextid++
      console.log "Client connected", id
      ws.on 'message', (message) ->
        try
          addRoutes id, JSON.parse(message)
        catch e
          console.log "Error parsing message", id, e
      ws.on 'close', (message) ->
        delRoutes id
        console.log "Client disconnected", id


Drop privileges
---------------

Drop privileges if we're root. We have to do this on nextTick so that the
event loop has time to set up sockets and things.

    process.nextTick ->
      if process.getuid() is 0
        process.setgid 'nobody'
        process.setuid 'nobody'
        throw new Error "Couldn't drop privileges" if process.getuid() is 0
