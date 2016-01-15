Devrouter Client
================

This client connects to the devrouter service and updates the routing table.
Generally the way you would use it is take your existing http serving command
(`npm start`, `http-server`, etc) and just prepend 'devroute' to the front.

    path = require 'path'
    WebSocket = require 'ws'
    freeport = require 'freeport-promise'
    {spawn} = require 'child_process'

    argv = require 'yargs'
    .option 'port',
      alias: 'p'
      describe: 'Port to route to'
      defaultDescription: 'random free port'

    .option 'name',
      alias: 'n'
      describe: 'Name to register with devrouter'
      default: -> path.basename(process.cwd())
      defaultDescription: 'current directory name'

    .option 'env',
      describe: 'Pass port as PORT environment variable'
      boolean: true
      default: true

    .usage 'Usage: $0 [options] [command [args...]]'
    .help 'help'
    .argv

    port = if argv.port then Promise.resolve(argv.port) else freeport()

    port.then (port) ->
      connect = ->
        ws = new WebSocket 'ws://127.0.0.1'
        ws.on 'open', ->
          route = {}
          route[argv.name] = port
          console.log "Route: #{argv.name}.dev => localhost:#{port}"
          ws.send JSON.stringify route

          run()

        ws.on 'close', -> reconnect 1000
        ws.on 'error', (e) -> reconnect 5000, e

        reconnect = (ms, e) ->
          console.error "devrouter connection failed. Reconnecting in #{ms} ms"
          console.error e if e
          setTimeout connect, ms

      connect()

    running = false
    run = ->
      return if running
      running = true
      process.env.PORT = port if argv.env
      if argv._.length > 0
        console.log "Spawning", argv._[0]
        spawn argv._[0], argv._.slice(1), stdio: 'inherit'
