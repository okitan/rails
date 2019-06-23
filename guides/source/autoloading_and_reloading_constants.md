**DO NOT READ THIS FILE ON GITHUB, GUIDES ARE PUBLISHED ON https://guides.rubyonrails.org.**

Autoloading and Reloading Constants (Zeitwerk Edition)
======================================================

This guide documents how autoloading and reloading works in `zeitwerk` mode.

After reading this guide, you will know:

* Autoloading configuration
* File system conventions
* Autoloading API

--------------------------------------------------------------------------------


Introduction
------------

INFO. This guide documents autoloading in `zeitwerk` mode. This is the new mode in Rails 6 applications by default. If you'd like to read about `classic` mode instead, please check [Autoloading and Reloading Constants (Classic Edition)][autoloading_and_reloading_constants_classic.html].

In a normal Ruby program, dependencies need to be loaded by hand. For example, the following controller uses classes `ApplicationController` and `Post`, and normally you'd need to put `require` calls for them:

```ruby
# DO NOT DO THIS.
require 'application_controller'
require 'post'
# DO NOT DO THIS.

class PostsController < ApplicationController
  def index
    @posts = Post.all
  end
end
```

This is not the case in Rails applications, where application classes and modules are just available everywhere

```ruby
class PostsController < ApplicationController
  def index
    @posts = Post.all
  end
end
```

Idiomatic Rails applications only issue `require` calls to load stuff from their `lib` directory, the Ruby standard library, Ruby gems, etc. That is, anything that does not belong to their autoload paths, explained next.

Configuration
-------------

INFO. You do not configure Zeitwerk manually in a Rails application. Rather, you configure the application as always, and Rails translates that to Zeitwerk on your behalf.

### Enabling Zeitwerk Mode

`zeitwerk` mode is enabled by default in Rails 6 applications running on CRuby:

```ruby
# config/application.rb
config.load_defaults "6.x" # enables zeitwerk mode in CRuby
```

In `zeitwerk` mode, Rails uses [Zeitwerk](https://github.com/fxn/zeitwerk) internally to autoload, reload, and eager load. Rails instantiates and configures a dedicated Zeitwerk instance that manages application code, and code of engines it might depend on.

### Opting Out

You can load Rails 6 defaults and still use the classic autoloader this way:

```ruby
# config/application.rb
config.load_defaults "6.x"
config.autoloader = :classic
```

That may be handy if upgrading to Rails 6 in different phases, but classic mode is discouraged for new applications.

`zeitwerk` mode is not available in versions of Rails previous to 6.0.

### Autoload paths

We call _autoload path_ to the absolute path of directories whose contents are to be autoloaded. For example, `#{Rails.root}/app/models`. Such directories represent the root namespace: `Object`.

INFO. Autoload paths are called _root directories_ in Zeitwerk parlance, but we'll stay with "autoload path" in this guide.

Within an autoload path, file names must match the constants they define as documented [here](https://github.com/fxn/zeitwerk#file-structure).

By default, the autoload paths of an application consist of all the subdirectories of `app` that exist when the application boots, plus the autoload paths of engines it might depend on.

For example, if `UsersHelper` is implemented in `app/helpers/users_helper.rb`, the module is autoloadable, you do not need (and should not write) a `require` call for it:

```
$ bin/rails runner 'p UsersHelper'
UsersHelper
```

It is important to stress that all the subdirectories of `app` belong to the autoload paths regardless of whether they are standard or not. For example, `app/controllers`, `app/models, etc., belong to the autoload paths, but any other existing custom subdirectories like `app/presenters`, `app/services`, etc., belong to the autoload paths too.

The list of autoload paths can be extended by mutating `config.autoload_paths`, in `config/application.rb`, but nowadays this is discouraged.

WARNING. Please, do not mutate `ActiveSupport::Dependencies.autoload_paths`, the public interface to change autoload paths is `config.autoload_paths`.

### Reloading

Rails automatically reloads classes and modules if application files change.

More precisely, if the web server is running and files have been modified, Rails unloads all autoloaded code just before the next request is processed. That way, application classes or modules used during that request are going to be autoloaded, thus picking up their current implementation in the file system.

Reloading can be enabled or disabled. The setting that controls this behavior is `config.cache_classes`, which is false by default in `development` mode (reloading enabled), and true by default in `production` mode (reloading disabled).

Rails detects files have changed using an evented file monitor (default), or walking the autoload paths. The technique depends on `config.file_watcher`.

In a Rails console there is no file watcher active regardless of the value of `config.cache_classes`. This is so because, normally, it would be confusing to have code reloaded in the middle of a console session, the same way you generally want an individual request to be served by a consistent, non-changing set of application classes and modules.

However, you can force a reload in the console executing `reload!`:

```
$ bin/rails c
Loading development environment (Rails 6.0.0)
irb(main):001:0> User.object_id
=> 70136277390120
irb(main):002:0> reload!
Reloading...
=> true
irb(main):003:0> User.object_id
=> 70136284426020
```

as you can see, the class objects stored in the `User` constant are different.

#### Reloading and stale class and module objects

It is very important to understand that Ruby does not have a way to truly reload classes and modules in memory, and have that reflected everywhere they are already used. In reality, "unloading" the `User` class means removing the `User` constant via `Object.send(:remove_const, "User")`.

For example, if an initializer stores and caches a certain class object

```ruby
# config/initializers/configure_payment_gateway.rb
# DO NOT DO THIS.
$PAYMENT_GATEWAY = Rails.env.production? ? RealGateway : MockedGateway
# DO NOT DO THIS.
```

and `MockedGateway` gets reloaded, `$PAYMENT_GATEWAY` has a stale class object, the original class object to which the constant `MockedGateway` evaluated to when the initializer ran.

Similarly, in the Rails console, if you have a user instance and reload:

```
> user = User.new
> reload!
```

the `user` object is instance of a stale class object. Ruby gives you a new class if you evaluate `User` again, but does not update the class `user` is instance of.

Another use case of this gotcha is subclassing reloadable classes in a place that is not reloaded:

```ruby
# lib/vip_user.rb
class VipUser < User
end
```

if `User` is reloaded, since `VipUser` is not, the superclass of `VipUser` is the original stale class object.

Bottom line: **do not cache reloadable classes or modules**.

### Eager Loading

### Single Table Inheritance

### Inflector

### zeitwerk:check
