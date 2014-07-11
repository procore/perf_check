## What is perf check

`perf_check` is a nice and easy way to benchmark branches of your rails app.

Imagine a rails-aware [apache ab](http://httpd.apache.org/docs/2.2/programs/ab.html). We mainly run it locally or on staging, to get a decent idea of how our branches might have affected app performance. Often, certain pages render differently if logged in, or admin, so `perf_check` provides an easy way to deal with that.

## How to install

Add to your Gemfile, probably just in the `:development` group

```
gem 'perf_check'
```

You will actually have to commit this. Preferably to master (as long as the gem exists on whatever reference branch you are testing against, you are good to go).

## How to use

Pick an url you want to bench

```
$ perf_check /notes/browse 
=============================================================================
PERRRRF CHERRRK! Grab a coffee and don't touch your working tree (we automate git)
=============================================================================


Benchmarking /notes/browse:
	Request  1:  774.8ms	  66MB
	Request  2:  773.4ms	  66MB
	Request  3:  771.1ms	  66MB
	Request  4:  774.1ms	  66MB
	Request  5:  773.7ms	  66MB
	Request  6:  774.8ms	  66MB
	Request  7:  773.4ms	  66MB
	Request  8:  771.1ms	  66MB
	Request  9:  774.1ms	  66MB
	Request  10: 773.7ms	  67MB

Benchmarking /notes/browse:
	Request  1:  20.2ms	  68MB
	Request  2:  23.0ms	  68MB
	Request  3:  19.9ms	  68MB
	Request  4:  19.5ms	  68MB
	Request  5:  19.4ms	  68MB
	Request  6:  20.2ms	  68MB
	Request  7:  23.0ms	  68MB
	Request  8:  19.9ms	  68MB
	Request  9:  19.5ms	  69MB
	Request  10: 19.4ms	  69MB

==== Results ====
/notes/browse
       master: 20.4ms
  your branch: 773.4ms
       change: +753.0ms (yours is 37.9x slower!!!)
```

## How does it work

In the above example, `perf_check`

* Launches its own rails server on a custom port (WITH CACHING FORCE-ENABLED)
* Hits /user/45/posts 11 times (throws away the first request)
* Git stashes in case you have uncommitted stuff. Checks out master. Restarts server
* Hits /user/45/posts 11 times (throws away the first request)
* Applies the stash if it existed
* Prints results

## Caveats of DOOOOM

### We automate git

**Do not edit the working tree while `perf_check` is running!** 

This program performs git checkouts and stashes, which are undone after the benchmark completes. If the working tree changes after the reference commit is checked out, numerous problems may arise. 

### We turn force caching on by default

Perf check start ups its rails server with `cache_classes=true` and `perform_caching=true` regardless of what's in your development.rb

You can pass `--clear-cache` which will run `Rails.cache.clear` before each batch of requests. This is useful when testing caching.

## benchmarking resources which require authorization

To enable benchmarking routes that require authorization, the rails app should provide a block to `PerfCheck::Server.authorization` that returns a cookie suitable for access to the route. For example, pop something like this in an initializer:

```Ruby
# config/initializers/perf_check.rb
if defined?(PerfCheck)
  PerfCheck::Server.authorization do |login, route|
      session = { :user_id => 1, :sesssion_id => '1234' }
      PerfCheck::Server.sign_cookie_data('_notes_session', session)
  end
end
```

Note that this logic depends greatly on your rails configuration. In this example we have assumed that the app uses an unencryped CookieStore to hold session data, and has a simple authorization filter such as

```Ruby
   before_filter :authorize
   def authorize
      return false unless session[:session_id]
      @user = User.find(session[:session_id])
   end
```

Alternatively, you can provide a block to `PerfCheck::Server.authorization_action`, which will replace the login action of your app with the given block. Requests will be made to this action when a session cookie is required. For example, for an app with `post '/login', :to => 'application#login'`, you could have:

```Ruby
# config/initializers/perf_check.rb
if ENV['PERF_CHECK']
   PerfCheck::Server.authorization_action(:post, '/login') do |login, route|
      session[:user_id] = User.find_by_login(login).id
   end
end
```

The `login` parameter to the authorization block is one of

  1. `:super`
  2. `:admin`
  3. `:standard`
  4. A user name, passed in as a string (corresponding to the `-u` command line argument)

It is up to the application to decide the semantics of this parameter.

The `route` parameter is a PerfCheck::TestCase. The block should return a cookie suitable for accessing `route.resource` (e.g. /users/1/posts) as the given `login`.

## command line usage
    Usage: perf_check [options] [route ...]
    Login options:
            --admin                      Log in as admin user for route
            --standard                   Log in as standard user for route
            --super                      Log in as super user
        -u, --user USER                  Log in as USER
        -L, --no-login                   Don't log in

    Benchmark options:
        -n, --requests N                 Use N requests in benchmark, defaults to
        -r, --reference COMMIT           Benchmark against COMMIT instead of master
        -q, --quick                      Fire off 5 requests just on this branch, no comparison with master

    Usage examples:
      Benchmark PostController#index against master
         perf_check /user/45/posts
         perf_check /user/45/posts -n5
         perf_check /user/45/posts --standard

      Benchmark against a specific commit
         perf_check /user/45/posts -r 0123abcdefg
         perf_check /user/45/posts -r HEAD~2

      Benchmark the changes in the working tree
         perf_check /user/45/posts -r HEAD

## example usage
```
$ perf_check -r HEAD -n5 /notes/browse 2>/dev/null
=============================================================================
PERRRRF CHERRRK! Grab a coffee and don't touch your working tree (we automate git)
=============================================================================


Benchmarking /notes/browse:
	Request  1: 774.8ms	  66MB
	Request  2: 773.4ms	  66MB
	Request  3: 771.1ms	  66MB
	Request  4: 774.1ms	  66MB
	Request  5: 773.7ms	  67MB



Benchmarking /notes/browse:
	Request  1: 20.2ms	  68MB
	Request  2: 23.0ms	  68MB
	Request  3: 19.9ms	  68MB
	Request  4: 19.5ms	  69MB
	Request  5: 19.4ms	  69MB

==== Results ====
/notes/browse
       master: 20.4ms
  your branch: 773.4ms
       change: +753.0ms (yours is 37.9x slower!!!)
```


### Troubleshooting

## Redis

Certain versions of redis might need the following snippet to fork properly:

```
  # Circumvent redis forking issues with our version of redis
  # Will be fixed when we update redis https://github.com/redis/redis-rb/issues/364
  class Redis
    class Client
      def ensure_connected
        tries = 0
        begin
          if connected?
            if Process.pid != @pid
              reconnect
            end
          else
            connect
          end
          tries += 1
          yield
        rescue ConnectionError
          disconnect
          if tries < 2 && @reconnect
            retry
          else
            raise
          end
        rescue Exception
          disconnect
          raise
        end
      end
    end
  end
  ```
