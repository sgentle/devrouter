Devrouter
=========

Devrouter is a simple tool for managing lots of local webservers for development. Do you ever get the problem where you have one project running on port 8000, one on 8080, one on 3000, etc and you can't keep track of them any more? That's the problem devrouter is designed to solve.

Instead of typing, `node myserver.js`, you can type `devroute node myserver.js`. Devrouter will find a free port, set it to the `PORT` environment variable so your server can pick it up. Meanwhile, it also passes that port and the name of the current directory to the Devrouter service's HTTP proxy. So you could now visit it at, for example, `http://myserver.dev`.

Now all your local development services can be referenced by nice names instead of arbitrary ports!


Setup
-----

Devrouter is on npm, so install it with `npm install -g devrouter`

You run the Devrouter service with `devrouter`, but you will probably need to `sudo devrouter` so that it can run as port 80. There are other, more secure ways of doing this, but they are all a bit annoying. We drop privileges after listening, so it's not so bad, but you may want to have a look through the code to make sure.

Once the Devrouter is running, you need to redirect some TLD to it. I suggest `.dev`. On OS X you can putting this in `/etc/resolver/dev`:

```
nameserver 127.0.0.1
port 10053
```

`.localhost` is another good choice, and I'm told it works by default on Linux. Unfortunately, Chrome interprets requests for `.localhost` in the URL bar as searches by default unless you prefix them with `http://`. Luckily, Google also registered the `.dev` TLD, presumably as a very expensive workaround. Thanks, Google!


Usage
-----

To use it, just type `devroute <command-here>`. Works great with http-server! `devroute http-server`

There are some more options you can check out with `devroute --help`

You can also put static routes in /etc/devrouter.json, they should look like this:
```
{
  "myserver": 1234
}
```
Which will permanently point myserver.dev (or myserver.localhost or whatever) to port 1234.


License
-------

MIT
