Bugsnag Notifier for Ruby
=========================

The Bugsnag Notifier for Ruby gives you instant notification of exceptions
thrown from your **Rails**, **Sinatra**, **Rack** or **plain Ruby** app.
Any uncaught exceptions will trigger a notification to be sent to your
Bugsnag project.

[Bugsnag](http://bugsnag.com) captures errors in real-time from your web,
mobile and desktop applications, helping you to understand and resolve them
as fast as possible. [Create a free account](http://bugsnag.com) to start
capturing exceptions from your applications.


Contents
--------

- [How to Install](#how-to-install)
- [Sending Custom Data With Exceptions](#sending-custom-data-with-exceptions)
- [Sending Handled Exceptions](#sending-handled-exceptions)
- [Configuration](#configuration)
- [Bugsnag Middleware](#bugsnag-middleware)
- [Deploy Tracking](#deploy-tracking)
- [EventMachine Apps](#eventmachine-apps)
- [Demo Applications](#demo-applications)

How to Install
--------------

1.  Add the `bugsnag` gem to your `Gemfile`

    ```ruby
    gem "bugsnag"
    ```

2.  Install the gem

    ```shell
    bundle install
    ```

3. Configure the Bugsnag module with your API key.

    **Rails**: Use our generator

    ```shell
    rails generate bugsnag YOUR_API_KEY_HERE
    ```

    **Other Ruby/Rack/Sinatra apps**: Put this snippet in your initialization.

    ```ruby
    Bugsnag.configure do |config|
      config.api_key = "YOUR_API_KEY_HERE"
    end
    ```

    The Bugsnag module will read the `BUGSNAG_API_KEY` environment variable if you
    do not configure one automatically.

4.  **Rack/Sinatra apps only**: Activate the Bugsnag Rack middleware

    ```ruby
    use Bugsnag::Rack
    ```

    **Sinatra**: Note that `raise_errors` must be enabled. If you are using custom
    error handlers, then you will need to notify Bugsnag explicitly:

    ```ruby
    error 500 do
      Bugsnag.auto_notify($!)
      erb :"errors/500"
    end
    ```

Sending Custom Data With Exceptions
-----------------------------------

It is often useful to send additional meta-data about your app, such as
information about the currently logged in user, along with any
exceptions, to help debug problems.

### Rails Apps

In any rails controller you can define a `before_bugsnag_notify` callback, which allows you to add this additional data by calling `add_tab` on the exception notification object. Please see the [Notification Object](#notification-object) for details on the notification parameter.

```ruby
class MyController < ApplicationController
  # Define the filter
  before_bugsnag_notify :add_user_info_to_bugsnag

  # Your controller code here

  private
  def add_user_info_to_bugsnag(notif)
    # Set the user that this bug affected
    # Email, name and id are searchable on bugsnag.com
    notif.user = {
      email: current_user.email,
      name: current_user.name,
      id: current_user.id
    }

    # Add some app-specific data which will be displayed on a custom
    # "Diagnostics" tab on each error page on bugsnag.com
    notif.add_tab(:diagnostics, {
      product: current_product.name
    })
  end
end
```

### Other Ruby Apps

In other ruby apps, you can provide lambda functions to execute before any `Bugsnag.notify` calls as follows. Don't forget to clear the callbacks at the end of each request or session. In Rack applications like Sinatra, this is automatically done for you.

```ruby
# Set a before notify callback
Bugsnag.before_notify_callbacks << lambda {|notif|
  notif.add_tab(:user_info, {
    name: current_user.name
  })
}

# Your app code here

# Clear the callbacks
Bugsnag.before_notify_callbacks.clear
```

### Notification Object

The notification object is passed to all [before bugsnag notify](#sending-custom-data-with-exceptions) callbacks and is used to manipulate the error report before it is transmitted.

#### add_tab

Call add_tab on a notification object to add a tab to the error report so that it would appear on your dashboard.

```ruby
notif.add_tab(:user_info, {
  name: current_user.name
})
```

The first parameter is the tab name that will appear in the error report and the second is the key, value list that will be displayed in the tab.

#### remove_tab

Removes a tab completely from the error report

```ruby
notif.remove_tab(:request)
```

#### ignore!

Calling ignore! on a notification object will cause the notification to not be sent to bugsnag. This means that you can choose dynamically not to send an error depending on application state or the error itself.

```ruby
notif.ignore! if foo == 'bar'
```

#### grouping_hash

Sets the grouping hash of the error report. All errors with the same grouping hash are grouped together. This is an advanced usage of the library and mis-using it will cause your errors not to group properly in your dashboard.

```ruby
notif.grouping_hash = "#{exception.message}#{exception.class}"
```

#### severity

Set the severity of the error. Severity can be `error`, `warning` or `info`.

```ruby
notif.severity = "error"
```

#### context

Set the context of the error report. This is notionally the location of the error and should be populated automatically. Context is displayed in the dashboard prominently.

```ruby
notif.context = "billing"
```

#### user

You can set or read the user with the user property of the notification. The user will be a hash of `email`, `id` and `name`.

```ruby
notif.user = {
  id: current_user.id,
  email: current_user.email,
  name: current_user.name
}
```

#### exceptions

Allows you to read the exceptions that will be combined into the report.

```ruby
puts "#{notif.exceptions.first.message} found!"
```

#### meta_data

Provides access to the meta_data in the error report.

```ruby
notif.ignore! if notif.meta_data[:sidekiq][:retry_count] > 2
```

### Exceptions with Meta Data

If you include the `Bugsnag::MetaData` module into your own exceptions, you can
associate meta data with a particular exception.

```ruby
class MyCustomException < Exception
  include Bugsnag::MetaData
end

exception = MyCustomException.new("It broke!")
exception.bugsnag_meta_data = {
  :user_info => {
    name: current_user.name
  }
}

raise exception
```

You can read more about how callbacks work in the
[Bugsnag Middleware](#bugsnag-middleware) documentation below.


Sending Handled Exceptions
----------------------------

If you would like to send non-fatal exceptions to Bugsnag, you can call
`Bugsnag.notify`:

```ruby
Bugsnag.notify(RuntimeError.new("Something broke"))
```

### Custom Data

You can also send additional meta-data with your exception:

```ruby
Bugsnag.notify(RuntimeError.new("Something broke"), {
  :user => {
    :username => "bob-hoskins",
    :registered_user => true
  }
})
```

### Severity

You can set the severity of an error in Bugsnag by including the severity option when
notifying bugsnag of the error,

```ruby
Bugsnag.notify(RuntimeError.new("Something broke"), {
  :severity => "error",
})
```

Valid severities are `error`, `warning` and `info`.

Severity is displayed in the dashboard and can be used to filter the error list.
By default all crashes (or unhandled exceptions) are set to `error` and all
`Bugsnag.notify` calls default to `warning`.

Rake Integration
----------------

Rake integration is automatically enabled in Rails 3/4 apps, so providing you load the environment
in your Rake tasks you dont need to do anything to get Rake support. If you choose not to load
your environment, you can manually configure Bugsnag with a `bugsnag.configure` block in the Rakefile.

Bugsnag can automatically notify of all exceptions that happen in your rake tasks. In order
to enable this, you need to `require "bugsnag/rake"` in your Rakefile, like so:

```ruby
require File.expand_path('../config/application', __FILE__)
require 'rake'
require "bugsnag/rake"

Bugsnag.configure do |config|
  config.api_key = "YOUR_API_KEY_HERE"
end

YourApp::Application.load_tasks
```

> Note: We also configure Bugsnag in the Rakefile, so the tasks that do not load the full
environment can still notify Bugsnag.

Standard Ruby Scripts
---------------------

If you are running a standard ruby script, you can ensure that all exceptions are sent to Bugsnag by
adding the following code to your app:

```ruby
at_exit do
  if $!
    Bugsnag.notify($!)
  end
end
```

Testing Integration
-------------------

To test that bugsnag is properly configured, you can use the test_exception rake task like this,

```bash
rake bugsnag:test_exception
```

A test exception will be sent to your bugsnag dashboard if everything is configured correctly.

Configuration
-------------

To configure additional Bugsnag settings, use the block syntax and set any
settings you need on the `config` block variable. For example:

```ruby
Bugsnag.configure do |config|
  config.api_key = "your-api-key-here"
  config.notify_release_stages = ["production", "development"]
end
```

###api_key

Your Bugsnag API key (required).

```ruby
config.api_key = "your-api-key-here"
```

###release_stage

If you would like to distinguish between errors that happen in different
stages of the application release process (development, production, etc)
you can set the `release_stage` that is reported to Bugsnag.

```ruby
config.release_stage = "development"
```

In rails apps this value is automatically set from `RAILS_ENV`, and in rack
apps it is automatically set to `RACK_ENV`. Otherwise the default is
"production".

###notify_release_stages

By default, we will notify Bugsnag of exceptions that happen in any
`release_stage`. If you would like to change which release stages
notify Bugsnag of exceptions you can set `notify_release_stages`:

```ruby
config.notify_release_stages = ["production", "development"]
```

###endpoint

By default, we'll send crashes to *notify.bugsnag.com* to display them on
your dashboard. If you are using *Bugsnag Enterprise* you'll need to set
this to be your *Event Server* endpoint, for example:

```ruby
config.endpoint = "bugsnag.example.com:49000"
```

###auto_notify

By default, we will automatically notify Bugsnag of any fatal exceptions
in your application. If you want to stop this from happening, you can set
`auto_notify`:

```ruby
config.auto_notify = false
```

###use_ssl

Enforces all communication with bugsnag.com be made via ssl. You can turn
this off if necessary.

```ruby
config.use_ssl = false
```

By default, `use_ssl` is set to true.

<!-- Custom anchor for linking from alerts -->
<div id="set-project-root"></div>
###project_root

We mark stacktrace lines as `inProject` if they come from files inside your
`project_root`. In rails apps this value is automatically set to `RAILS_ROOT`,
otherwise you should set it manually:

```ruby
config.project_root = "/var/www/myproject"
```

###app_version

If you want to track which versions of your application each exception
happens in, you can set `app_version`. This is set to `nil` by default.

```ruby
config.app_version = "2.5.1"
```

###params_filters

Sets which keys should be filtered out from `params` hashes before sending
them to Bugsnag. Use this if you want to ensure you don't send sensitive data
such as passwords, and credit card numbers to our servers. You can add both
strings and regular expressions to this array. When adding strings, keys which
*contain* the string will be filtered. When adding regular expressions, any
keys which *match* the regular expression will be filtered.

```ruby
config.params_filters += ["credit_card_number", /^password$/]
```

By default, `params_filters` is set to `["password", "secret"]`, and for rails
apps, imports all values from `Rails.configuration.filter_parameters`.

###ignore_classes

Sets for which exception classes we should not send exceptions to bugsnag.com.

```ruby
config.ignore_classes << "ActiveRecord::StatementInvalid"
```

You can also provide a lambda function here to ignore by other exception
attributes or by a regex:

```ruby
config.ignore_classes << lambda {|ex| ex.message =~ /timeout/}
```

By default, `ignore_classes` contains the following:

```ruby
[
  "ActiveRecord::RecordNotFound",
  "ActionController::RoutingError",
  "ActionController::InvalidAuthenticityToken",
  "CGI::Session::CookieStore::TamperedWithCookie",
  "ActionController::UnknownAction",
  "AbstractController::ActionNotFound"
]
```

###ignore_user_agents

Sets an array of Regexps that can be used to ignore exceptions from
certain user agents.

```ruby
config.ignore_user_agents << %r{Chrome}
```

By default, `ignore_user_agents` is empty, so exceptions caused by all
user agents are reported.

###proxy_host

Sets the address of the HTTP proxy that should be used for requests to bugsnag.

```ruby
config.proxy_host = "10.10.10.10"
```

###proxy_port

Sets the port of the HTTP proxy that should be used for requests to bugsnag.

```ruby
config.proxy_port = 1089
```

###proxy_user

Sets the user that should be used to send requests to the HTTP proxy for requests to bugsnag.

```ruby
config.proxy_user = "proxy_user"
```

###proxy_password

Sets the password for the user that should be used to send requests to the HTTP proxy for requests to bugsnag.

```ruby
config.proxy_password = "proxy_secret_password_here"
```

###timeout
By default the timeout for posting errors to Bugsnag is 5 seconds, to change this
you can set the `timeout`:

```ruby
config.timeout = 10
```

###logger

Sets which logger to use for Bugsnag log messages. In rails apps, this is
automatically set to use `Rails.logger`, otherwise it will be set to
`Logger.new(STDOUT)`.

###middleware

Provides access to the middleware stack, see the
[Bugsnag Middleware](#bugsnag-middleware) section below for details.

###app_type

You can set the type of application executing the current code by using `app_type`:

```ruby
config.app_type = "resque"
```

This is usually used to represent if you are running in a Rails server, Sidekiq job or
Rake task for example. Bugsnag will automatically detect most application types for you.

###send_environment

Bugsnag can transmit your rack environment to help diagnose issues. This environment
can sometimes contain private information so Bugsnag does not transmit by default. To
send your rack environment, set the `send_environment` option to `true`.

```ruby
config.send_environment = true
```

Bugsnag Middleware
------------------

The Bugsnag Notifier for Ruby provides its own middleware system, similar to
the one used in Rack applications. Middleware allows you to execute code
before and after an exception is sent to bugsnag.com, so you can do things
such as:

-   Send application-specific information along with exceptions, eg. the name
    of the currently logged in user,
-   Write exception information to your internal logging system.

To make your own middleware, create a class that looks like this:

```ruby
class MyMiddleware
  def initialize(bugsnag)
    @bugsnag = bugsnag
  end

  def call(notification)
    # Your custom "before notify" code

    @bugsnag.call(notification)

    # Your custom "after notify" code
  end
end
```

You can then add your middleware to the middleware stack as follows:

```ruby
Bugsnag.configure do |config|
  config.middleware.use MyMiddleware
end
```

You can also view the order of the currently activated middleware by running `rake bugsnag:middleware`.

Check out Bugsnag's [built in middleware classes](https://github.com/bugsnag/bugsnag-ruby/tree/master/lib/bugsnag/middleware)
for some real examples of middleware in action.

### Multiple projects

If you want to divide errors into multiple Bugsnag projects, you can specify the API key as a parameter to `Bugsnag.notify`:

```ruby
rescue => e
  Bugsnag.notify e, api_key: "your-api-key-here"
end
```

### Grouping hash

If you want to override Bugsnag's grouping algorithm, you can specify a grouping hash key as a parameter to `Bugsnag.notify`:

```ruby
rescue => e
  Bugsnag.notify e, grouping_hash: "this-is-my-grouping-hash"
end
```

All errors with the same groupingHash will be grouped together within the bugsnag dashboard.

Deploy Tracking
---------------

Bugsnag allows you to track deploys of your apps. By sending the
source revision or application version to bugsnag.com when you deploy a new
version of your app, you'll be able to see which deploy each error was
introduced in.

### Using Capistrano

If you use [capistrano](https://github.com/capistrano/capistrano) to deploy
your apps, you can enable deploy tracking by adding the integration to your
app's `deploy.rb`:

```ruby
require "bugsnag/capistrano"

set :bugsnag_api_key, "api_key_here"
```

### Using Rake

If you aren't using capistrano, you can run the following rake command from
your deploy scripts.

```shell
rake bugsnag:deploy BUGSNAG_REVISION=source-control-revision BUGSNAG_RELEASE_STAGE=production BUGSNAG_API_KEY=api-key-here
```

The bugsnag rake tasks will be automatically available for Rails 3/4
apps, to make the rake tasks available in other apps, add the following to
your `Rakefile`:

```ruby
require "bugsnag/tasks"
```

### Configuring Deploy Tracking

You can set the following environmental variables to override or specify
additional deploy information:

-   **BUGSNAG_API_KEY** -
    Your Bugsnag API key (required).
-   **BUGSNAG_RELEASE_STAGE** -
    The release stage (eg, production, staging) currently being deployed.
    This is set automatically from your Bugsnag settings or rails/rack
    environment.
-   **BUGSNAG_REPOSITORY** -
    The repository from which you are deploying the code. This is set
    automatically if you are using capistrano.
-   **BUGSNAG_BRANCH** -
    The source control branch from which you are deploying the code.
    This is set automatically if you are using capistrano.
-   **BUGSNAG_REVISION** -
    The source control revision for the code you are currently deploying.
    This is set automatically if you are using capistrano.
-   **BUGSNAG_APP_VERSION** -
    The app version of the code you are currently deploying. Only set this
    if you tag your releases with [semantic version numbers](http://semver.org/)
    and deploy infrequently.

For more information, check out the [deploy tracking api](https://bugsnag.com/docs/deploy-tracking-api)
documentation.

### EventMachine Apps

If your app uses [EventMachine](http://rubyeventmachine.com/) you'll need to
manually notify Bugsnag of errors. There are two ways to do this in your
EventMachine apps, first you should implement `EventMachine.error_handler`:

```ruby
EventMachine.error_handler{|e|
  Bugsnag.notify(e)
}
```

If you want more fine-grained error handling, you can use the
[errback](http://eventmachine.rubyforge.org/EventMachine/Deferrable.html#errback-instance_method)
function, for example:

```ruby
EventMachine::run do
  server = EventMachine::start_server('0.0.0.0', PORT, MyServer)
  server.errback {
    EM.defer do
      Bugsnag.notify(RuntimeError.new("Something bad happened"))
    end
  }
end
```

For this to work, include [Deferrable](http://eventmachine.rubyforge.org/EventMachine/Deferrable.html)
in your `MyServer`, then whenever you want to raise an error, call `fail`.

### Integrations

Bugsnag ruby works out of the box with Rails, Sidekiq, Resque, DelayedJob (3+), Mailman, Rake and Rack. It
should be easy to add support for other frameworks, either by sending a pull request here or adding a hook
to those projects.

Demo Applications
-----------------

[There are demo applications that use the Bugsnag Ruby gem](https://github.com/bugsnag/bugsnag-example-apps/tree/master/apps/ruby):
examples include Rails, Sinatra, Rack, Padrino integrations, etc.

Reporting Bugs or Feature Requests
----------------------------------

Please report any bugs or feature requests on the github issues page for this
project here:

<https://github.com/bugsnag/bugsnag-ruby/issues>


Contributing
------------

-   [Fork](https://help.github.com/articles/fork-a-repo) the [notifier on github](https://github.com/bugsnag/bugsnag-ruby)
-   Commit and push until you are happy with your contribution
-   Run the tests with `rake spec` and make sure they all pass
-   [Make a pull request](https://help.github.com/articles/using-pull-requests)
-   Thanks!


Build Status
------------
[![Build Status](https://secure.travis-ci.org/bugsnag/bugsnag-ruby.png)](http://travis-ci.org/bugsnag/bugsnag-ruby)


License
-------

The Bugsnag ruby notifier is free software released under the MIT License.
See [LICENSE.txt](https://github.com/bugsnag/bugsnag-ruby/blob/master/LICENSE.txt) for details.
