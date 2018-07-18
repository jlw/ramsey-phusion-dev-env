# A nice development environment with Phusion Passenger #

## Background (why not just `bundle exec rails s`?) ##

### TL;DR ###

**Pros**

- allows for simpler configuration and access of multiple concurrently-running apps
- removes the need to keep a terminal session for each running app
- eliminates the need for port 3000 (and conflicts when running multiple apps)
- allow access from mobile devices and VMs (i.e. test IE / Edge) in development with xip.io

**Cons**

- additional setup with a (low) risk of breaking in the future
- sorting out configuration errors could be a problem for devs without experience managing nginx or Apache
- xip.io addresses are subject to change with every DHCP assignment (possibly multiple times a day)


### Story time ###

Switching back and forth between a number of web apps has been a reality of work for most of my career; more often than not, I will work on multiple apps during any given week, and occasionally several within a single day. As a result, I find that the balance of developer happiness on one side and more configuration and moving parts on the other side changes as the number of apps increases.

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

The recent rollout of single sign-on and the corresponding need to use hard-coded domains in an environment where `.test` domain names are not universally used in development has spurred me to return to the higher-configuration-overhead of local persistent web and app servers. This time, however, Passenger has added nginx compatibility, so I can use a server that I prefer for its performance, lower overhead, and configuration style (and "Web Sharing" appears to no longer be a thing in macOS - _c'est la vie_).

_Note:_ Passenger has both open source and enterprise editions - the open source option is no longer immediately obvious on their site.


## Prerequisites ##

1. Get a Mac, or a different set of installation instructions.
2. If you don't already have [Homebrew][homebrew] installed, _are you crazy‽_
3. If you have another web server already running on port 80 locally, these instructions probably won't work for you. If you're running Apache, you can find comparable instructions on the [Passenger][passenger] site.
4. If you have puma-dev installed, you'll need to uninstall it first:
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
  server_name *.id.ramseysolutions.localhost ~^\w+\.id\.ramseysolutions[\.\d]+\.xip\.io$;
  root /Users/jer/Sites/ramsey/access-control-app/public;
  passenger_enabled on;
  passenger_ruby /Users/jer/.rbenv/shims/ruby;
  passenger_sticky_sessions on;
  passenger_app_env development;
}
```
For most apps, like `fpu-online`, the server name declaration is a bit simpler: `server_name local.financialpeace.com ~^financialpeace[\.\d]+\.xip\.io$;`.

_Notes:_

- You will need to restart nginx after making configuration changes: `sudo brew services restart nginx`
- I use [rbenv][rbenv], so if you are still using RVM, your ruby path will differ from mine. _I am grateful to the creator of RVM - he provided tools that were desperately needed - but I prefer rbenv's Unix-style architecture of multiple smaller utilities and I find rbenv's shims to be a saner solution than RVM's method of commandeering my shell.__


### Apps ###

Most, if not all, Ramsey apps will require some alternate configuration to run with xip.io for mobile/VM testing. My current `fpu-online/.env.development.local` looks like this:
```
APP_URL=http://financialpeace.10.0.1.2.xip.io
ASSET_HOST=http://financialpeace.10.0.1.2.xip.io
SSO_PROVIDER_URL=http://financialpeace.id.ramseysolutions.10.0.1.2.xip.io
```


### /etc/hosts ###

For apps that need addresses outside of `*.test` and xip.io, you will need to manually create each entry with `sudo vi /etc/hosts` or e.g. `echo "127.0.0.1	local.financialpeace.com" | sudo tee -a /etc/hosts`.


[homebrew]: https://brew.sh/
[passenger]: https://www.phusionpassenger.com/
[passenger-troubleshooting-guide]: https://www.phusionpassenger.com/library/admin/nginx/troubleshooting/ruby/
[rbenv]: https://github.com/rbenv/rbenv
