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
    bouncy = require 'bouncy'

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

This router passes connections through to whatever ports are defined in the registry.

Handle errors in the routed connection (eg server not up)

    handleError = (res) -> (e) ->
      res.statusCode = 500
      res.end JSON.stringify e

Our homepage is just a list of routed names/ports.

    home = (req, res) ->
      return if req.headers.connection is 'Upgrade'
      return res.end ("#{name}.dev => localhost:#{port}" for name, {port} of registry).join '\n'

The router itself goes to the homepage on the root domain ie 'dev.' or
'localhost' or '127.0.0.1'. For anything else it looks up the root domain in
our registry and routes the request there.

    router = bouncy (req, res, bounce) ->
      return home(req, res) if !req.headers.host or req.headers.host.match /^[\d.]+$/

      hostParts = req.headers.host.split '.'
      hostParts.pop() if hostParts[hostParts.length-1] is ''
      console.log "hostParts", hostParts
      return home(req, res) unless hostParts.length > 1

      name = hostParts[hostParts.length-2]
      if port = registry[name]
        s = bounce port
        s.on 'error', handleError res
      else
        res.statusCode = 404
        res.end "#{name} not found"

    router.listen httpPort, host

Drop privileges if we're root.

    process.nextTick ->
      if process.getuid() is 0
        process.setgid 'nobody'
        process.setuid 'nobody'
        throw new Error "Couldn't drop privileges" if process.getuid() is 0

Websocket Controller
--------------------

We use this to add routes. It attaches to the same HTTP server we use for
routing.

The design is that the routes are kept alive by an active websocket
connection, and removed when that connection terminates. The only exception
are routes added within the config file at startup.

    nextid = 0
    wss = new WebSocketServer server: router
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

