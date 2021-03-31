# Setting up a development environment on my M1 MacBook Air
As a challenge to myself to keep my M1 Mac as "pure-ARM" as possible, I will attempt to set up everything without requiring Rosetta 2.

## Goals
- [x] Homebrew
- [x] PHP 7.4 and PHP-FPM
- [x] Composer 1.x
- [x] Node.js 14 LTS
- [x] Yarn 1.x
- [x] Heroku CLI
- [ ] Zerotier
- [ ] PostgresSQL 11.x
- [ ] Redis
- [ ] Nginx
- [ ] Logrotate for all the various logfiles
- [x] Cocoapods
- [ ] iOS: getting bundled static libraries in my iOS projects to link for ARM iOS simulator

### Homebrew
Difficulty: Easy

Following the setup instructions at https://brew.sh results in a working Homebrew installation at `/opt/homebrew`. 

As I prefer to install versioned formulae for my development tools (e.g. `php@7.4`, `node@14`, `postgresql@11`) I need to manually place their binaries' locations in my path. I do that by placing the following in my `.zprofile`:
```
eval "$(/opt/homebrew/bin/brew shellenv)"

# Add Homebrew's versioned formula paths
path=("/Users/kit/.bin"
  "$(/opt/homebrew/bin/brew --prefix php@7.4)/sbin"
  "$(/opt/homebrew/bin/brew --prefix php@7.4)/bin"
  "$(/opt/homebrew/bin/brew --prefix node@14)/bin"
  "$(/opt/homebrew/bin/brew --prefix ruby@2.7)/bin"
  "/opt/homebrew/lib/ruby/gems/2.7.0/bin"
  $path
)
```

### PHP 7.4 and PHP-FPM
Difficulty: Easy

Installed without any issues using `brew install php@7.4`.

By default, FPM is configured to listen on port 9000. I prefer PHP-FPM listening on a UNIX socket, so I edited the FPM `www` pool configuration file at `/opt/homebrew/etc/php/7.4/php-fpm.d`. I changed the `listen` directive to 
```
; - 'listen' (unixsocket)
;   'ip.add.re.ss:port'    - to listen on a TCP socket to a specific IPv4 address on
;   '[ip:6:addr:ess]:port' - to listen on a TCP socket to a specific IPv6 address on
;   'port'                 - to listen on a TCP socket to all addresses
;   '/path/to/unix/socket' - to listen on a unix socket.
listen = /opt/homebrew/var/run/php-fpm.sock
```

After that, starting PHP-FPM as a service is done with `brew services start php@7.4`. 

### Composer 1.x
Difficulty: Easy

I installed Composer to `~/.bin` by downloading `composer-setup.php` from https://getcomposer.org/download/ and running
```
$ php composer-setup.php --install-dir=~/.bin --filename=composer --version=1.10.20
```

After the install, I added `~/.bin` to my PATH (see the Homebrew section above).

### Node.js 14 LTS
Difficulty: Easy

```
brew install node@14
```

### Yarn 1.x
Difficulty: Easy

Yarn is now on version 2.x but I require Yarn 1.x installed. No biggie, Yarn 2.x is now still an "optional" install, the default installation will yield the latest 1.x version. I go to https://yarnpkg.com/getting-started/install and install Yarn via NPM with
```
$ npm install -g yarn
```

### Heroku CLI
Difficulty: Moderate

Following the instructions at https://devcenter.heroku.com/articles/heroku-cli#other-installation-methods for installing via Homebrew and using the standalone curl-ed script did not work. This is because Heroku does not provide a downloadable build for `darwin-arm`. 

However, the Heroku CLI is a Node.js CLI app, so it shouldn't be too hard to get working. I downloaded the macOS tarball and manually extracted it to `/usr/local/lib`. The tarball contains a Node binary that the CLI uses as a runtime, but this was built for x86_64 so it will not run here. It turns out that the Heroku folks have considered this possibility: the standalone install script contains this line:
```
  # on alpine (and maybe others) the basic node binary does not work
  # remove our node binary and fall back to whatever node is on the PATH
  /usr/local/lib/heroku/bin/node -v || rm /usr/local/lib/heroku/bin/node
```

Cool! I deleted the bundled node binary and also symlinked the entrypoint into `~/.bin`, and tested:
```
$ rm /usr/local/lib/heroku/bin/node
$ ln -s /usr/local/lib/heroku/bin/heroku ~/.bin/heroku

$ heroku --version
heroku/7.51.0 darwin-arm64 node-v14.16.0
```

### Cocoapods
Difficulty: Moderate

Cocoapods does not have official support for Apple Silicon yet, see https://github.com/CocoaPods/CocoaPods/issues/9907. However, from my testing, most of the bugs in Cocoapods' upstream dependencies have been fixed. Unfortunately, some of the patches are in the dependencies' `master` branch but no release has been cut. 

As Apple has deprecated built-in scripting runtimes in macOS, I got started by installing `ruby@2.7` from Homebrew. The other benefit of installing your own copy of Ruby is that I do not need to use `sudo` to install Cocoapods. If I were using the macOS-provided Ruby, installing global gems will require `sudo`. 

```
$ brew install ruby@2.7
```

After Ruby is installed, I have to place the gem bin directory `/opt/homebrew/lib/ruby/gems/2.7.0/bin` into my PATH.

```
$ gem install cocoapods
```

At this point, `pod --version` should be working.

```
$ pod --version
1.10.1
```

However, running `pod install` or `pod update` in my projects result in an segfault in the `ethon` library (a Cocoapods dependency, `typhoeus`, uses `ethon` to interface with `curl` apparently:
```
/opt/homebrew/lib/ruby/gems/2.7.0/gems/ethon-0.12.0/lib/ethon/curls/infos.rb:127: [BUG] Segmentation fault at 0x0000000000000002
```

I found this [issue](https://github.com/typhoeus/typhoeus/issues/667) in the Typhoeus repository, which provided a nice two-liner test case:
```rb
require 'typhoeus'
Typhoeus.get("www.example.com")
```

The issue also mentioned that a [patch](https://github.com/typhoeus/ethon/commit/3e871407463abd3c36a0fcef865bb9ae1ccb92a9) has been released for `ethon`. Unfortunately, `ethon`'s latest [release](https://github.com/typhoeus/ethon/releases) dates back to Jan 2019 😫 so I had to figure out how to install the `HEAD` version of the `ethon` gem.

After some googling, I chanced upon [`specific_install`](https://github.com/rdp/specific_install), which allowed me to install "edge" gems from a Github repository.

```
$ gem install specific_install
$ gem specific_install https://github.com/typhoeus/ethon
```
The test case worked after this, and `pod install` also worked!

On a related note, Cocoapods is moving away from these libraries in a bid to make it ARM-compatible: https://github.com/CocoaPods/CocoaPods/issues/10408. It is possible that the above workarounds will not be needed in the future.

## Other notes
### Laravel Mix
Mix/Webpack works, but crashes at the end of the run when attempting to send the completed macOS notification. 
```
 DONE  Compiled successfully in 78066ms                                                                                 10:05:44 AM

Error: spawn Unknown system error -86
    at ChildProcess.spawn (internal/child_process.js:403:11)
    at spawn (child_process.js:553:9)
    at Object.execFile (child_process.js:237:17)
    at Object.module.exports.fileCommandJson (/Users/kit/example-project/node_modules/node-notifier/lib/utils.js:83:13)
    at NotificationCenter.notify (/Users/kit/example-project/node_modules/node-notifier/notifiers/notificationcenter.js:81:11)
    at module.exports.WebpackNotifierPlugin.compilationDone (/Users/kit/example-project/node_modules/webpack-notifier/index.js:78:18)
    at AsyncSeriesHook.eval [as callAsync] (eval at create (/Users/kit/example-project/node_modules/tapable/lib/HookCodeFactory.js:33:10), <anonymous>:36:1)
    at AsyncSeriesHook.lazyCompileHook (/Users/kit/example-project/node_modules/tapable/lib/Hook.js:154:20)
    at /Users/kit/example-project/node_modules/webpack/lib/Compiler.js:304:22
    at Compiler.emitRecords (/Users/kit/example-project/node_modules/webpack/lib/Compiler.js:499:39)
    at /Users/kit/example-project/node_modules/webpack/lib/Compiler.js:298:10
    at /Users/kit/example-project/node_modules/webpack/lib/Compiler.js:485:14
    at AsyncSeriesHook.eval [as callAsync] (eval at create (/Users/kit/example-project/node_modules/tapable/lib/HookCodeFactory.js:33:10), <anonymous>:22:1)
    at AsyncSeriesHook.lazyCompileHook (/Users/kit/example-project/node_modules/tapable/lib/Hook.js:154:20)
    at /Users/kit/example-project/node_modules/webpack/lib/Compiler.js:482:27
    at /Users/kit/example-project/node_modules/neo-async/async.js:2818:7
    at done (/Users/kit/example-project/node_modules/neo-async/async.js:3522:9)
    at AsyncSeriesHook.eval [as callAsync] (eval at create (/Users/kit/example-project/node_modules/tapable/lib/HookCodeFactory.js:33:10), <anonymous>:6:1)
    at /Users/kit/example-project/node_modules/webpack/lib/Compiler.js:464:33
    at /Users/kit/example-project/node_modules/graceful-fs/graceful-fs.js:61:14
    at FSReqCallback.oncomplete (fs.js:171:23)
error Command failed with exit code 1.
info Visit https://yarnpkg.com/en/docs/cli/run for documentation about this command.
```

This is because `terminal-notifier` has not been updated for Apple Silicon: https://github.com/mikaelbr/node-notifier/issues/361. The workaround is simply to disable notifications in `webpack.mix.js`:

```js
if (process.arch === 'arm64') mix.disableNotifications();
```

## Mac apps on watchlist
The following apps I use either do not have Apple Silicon support, or have been built for ARM64 but is buggy.
- Spotify (no Apple Silicon version)
- Fork.app (Apple Silicon build crashes upon opening any Git repository - sent bug report)
