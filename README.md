Devrouter
=========

Devrouter is a simple tool for managing lots of local webservers for development. Do you ever get the problem where you have one project running on port 8000, one on 8080, one on 3000, etc and you can't keep track of them any more? That's the problem devrouter is designed to solve.

Instead of typing, `node myserver.js`, you can type `devroute node myserver.js`. Devrouter will find a free port and set it to the `PORT` environment variable so your server can pick it up. Meanwhile, it also passes that port and the name of the current directory to the Devrouter service's HTTP proxy. So you can now visit it at, for example, `http://myserver.dev`.

Now all your local development services can be referenced by nice names instead of arbitrary ports!


Setup
-----

Devrouter is on npm, so install it with `npm install -g devrouter`

Devrouter has two commands. The service is `devrouter` and the utility is `devroute` <-- note the lack of an `r` at the end.

You run the Devrouter service with `devrouter`, but you will probably need to `sudo devrouter` so that it can run as port 80. There are other, more secure ways of doing this, but they are all a bit annoying. We drop privileges after listening, so it's not so bad, but you may want to have a look through the code to make sure.

If you want it to run at startup, you can copy [the plist file](https://github.com/sgentle/devrouter/blob/master/com.samgentle.devrouter.plist) into /Library/LaunchDaemons and start it with `sudo launchctl load /Library/LaunchDaemons/com.samgentle.devrouter.plist`

Once the Devrouter is running, you need to redirect some TLD to it. I suggest `.dev`. On OS X you can do that by putting this in `/etc/resolver/dev`:

```
nameserver 127.0.0.1
port 10053
```

`.localhost` is another good choice, and I'm told it works by default on Linux. Unfortunately, Chrome interprets requests for `.localhost` in the URL bar as searches by default unless you prefix them with `http://`. Luckily, Google also registered the `.dev` TLD, presumably as a very expensive workaround. Thanks, Google!


Usage
-----

To use it, just type `devroute <command-here>`. Works great with [http-server](https://github.com/indexzero/http-server)! `devroute http-server`

There are some more options you can check out with `devroute --help`, including setting a specific port and name

You can also put static routes in /etc/devrouter.json, they should look like this:
```
{
  "myserver": 1234
}
```
Which will permanently point myserver.dev (or myserver.localhost or whatever) to port 1234.


tl:dr (OSX)
----------

```
npm install -g devrouter
sudo devrouter
sudo sh -c 'echo nameserver 127.0.0.1\\nport 10053 > /etc/resolver/dev'
cd my-project
devroute ./thing_that_runs_my_server
open http://my-project.dev
```


License
-------

MIT
