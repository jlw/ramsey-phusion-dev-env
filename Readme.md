# A nice development environment with Phusion Passenger #

## Background (why not just `bundle exec rails s`?) ##

I have found myself switching back and forth between a number of web apps on many days, and more often than not, I will work on multiple apps within a given week. As a result, I find that balance of developer happiness on one side and more configuration and moving parts on the other side changes as the number of apps increases.

The default of managing `rails server` (bound to a specific port that conflicts with other apps' usage of `rails server`) in a terminal session that may be one of many open is great for getting developers up and running quickly. It is not so great if I want to quickly check on another app's work in progress or if I forgot to stop the running server when I switched contexts to another app.

The default does not work at all in cases where I need to work with multiple apps running on localhost (e.g. a site that is composed of multiple separate apps and services).

I used Apache + [Phusion Passenger][passenger] to deploy my first production Rails app over a decade ago, and quickly set up the same environment for development. I found Passenger to be quite stable, and when I needed to reboot the app I could type `touch tmp/restart.txt` rather than `^C` followed by `rails s` and waiting a moment to make sure the app finishes booting before refreshing my browser.

When [pow](http://pow.cx/) was released it was a revelation for a number of reasons:

- no need to manage domains in `/etc/hosts`;
- no need to manually reboot apps (pow watched core Rails files for changes and proactively rebooted the app when they changed);
- no need to have Apache running locally;
- no lost time when an Apple system updated broke my Apache configuration; and
- including `xip.io` domains for proper mobile browser testing.

When pow became flaky (for me, at least), I switched to [puma-dev](https://github.com/puma/puma-dev) which is practically a drop-in replacement for pow.

The recent rollout of single sign-on and corresponding need to use hard-coded domains in an environment where `.test` domain names are not universally used in development has spurred me to return to the higher-configuration-overhead of . This time, however, Passenger has added nginx compatibility, so I can use a server that I prefer for its performance, lower overhead, and configuration style (and "Web Sharing" appears to no longer be a thing in macOS - _c'est la vie_).

_Note:_ Passenger has both open source and enterprise editions - the open source option is no longer immediately obvious on their site.


## Prerequisites ##

1. If you don't already have [Homebrew][homebrew] installed, _are you crazy‽_
2. If you have another web server already running on port 80 locally, these instructions probably won't work for you. If you're running Apache, you can find comparable instructions on the [Passenger][passenger] site.
3. If you have puma-dev installed, you'll need to uninstall it first:
```
puma-dev -cleanup
puma-dev -d test -cleanup
puma-dev -uninstall
puma-dev -d test -uninstall
```


## Installation ##


### dnsmasq _(optional)_ ###

With dnsmasq, we can replicate part of pow/puma-dev by routing all DNS requests for e.g. `*.test` to localhost. By using dnsmasq, I can skip the step of editing `/ect/hosts` for each site I setup locally, plus I find `http://example.test` to be more honest and aesthetically pleasing than e.g. `http://local.example.com`.

`brew install dnsmasq`

Edit `/usr/local/etc/dnsmasq.conf`, find the "address" section (currently around line 80), and add a new line with `address=/.test/127.0.0.1`.

```
sudo cp -v $(brew --prefix dnsmasq)/homebrew.mxcl.dnsmasq.plist /Library/LaunchDaemons
sudo launchctl load -w /Library/LaunchDaemons/homebrew.mxcl.dnsmasq.plist
sudo mkdir /etc/resolver
sudo bash -c 'echo "nameserver 127.0.0.1" > /etc/resolver/test'
```


### nginx + Passenger ###

`brew install nginx --with-passenger`

This creates configuration files in `/usr/local/etc/nginx/`, which could be overwritten by a future upgrade, so create your own copy to store your changes (I create a git repo) and symlink it to that location. To begin with, edit `nginx.conf` to change `#user nobody;` to `user <your user name> staff;`

Passenger is installed, but still needs to be enabled inside nginx. Run `brew info nginx --with-passenger` and follow the instructions. I use rbenv, so I changed the path to the rbenv ruby shim (`~/.rbenv/shims/ruby`).

Ensure that passenger is installed correctly:
`sudo /usr/local/bin/passenger-config validate-install`

Set nginx to always run (as a background service, with sudo so it can listen on port 80):
`sudo brew services start nginx`

Check that nginx has started the Passenger core processes - look for both nginx and Passenger processes listed in the output of
`sudo /usr/local/bin/passenger-memory-stats`

If that fails, check the [troubleshooting guide][passenger-troubleshooting-guide].


## Configuration ##


### nginx ###

The configuration files are located in `/usr/local/etc/nginx/` and will almost certainly be overwritten, at least in part, by future upgrades. I create my own copy to store my changes (in a git repo, naturally), and symlink it to that path.

To start with, I change `#user nobody;` to `user <your user name> staff;` to prevent permissions errors down the road. Next, I add the following block within the `http {…}` block so that nginx responds with a sample page on port 80:
```
    server {
        listen       80;
        server_name  localhost;

        location / {
            root   html;
            index  index.html index.htm;
        }

        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }
    }
```

Next, we'll add a `servers/*.conf` file for each app that we want to run locally. Here's my copy of `servers/access-control-app.conf`:
```
server {
  listen 80;
  server_name *.id.ramseysolutions.localhost;
  root /Users/jer/Sites/ramsey/access-control-app/public;
  passenger_enabled on;
  passenger_ruby /Users/jer/.rbenv/shims/ruby;
  passenger_sticky_sessions on;
  passenger_app_env development;
}
```

_Note:_ I use [rbenv][rbenv], so if you are still using RVM, your ruby path will differ from mine. (I am grateful to the creator of RVM - he provided tools that were desperately needed - but I find rbenv to be a much saner solution than RVM's method of commandeering my shell.)


### /etc/hosts ###

For apps that need addresses outside of `*.test`, you will need to manually create each entry with `sudo vi /etc/hosts` or e.g. `echo "127.0.0.1	local.financialpeace.com" | sudo tee -a /etc/hosts`.


[homebrew]: https://brew.sh/
[passenger]: https://www.phusionpassenger.com/
[passenger-troubleshooting-guide]: https://www.phusionpassenger.com/library/admin/nginx/troubleshooting/ruby/
[rbenv]: https://github.com/rbenv/rbenv
