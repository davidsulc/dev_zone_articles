Having gone over the basics in a previous article (...), we will now cover how you can get a basic Rails 3 app (using the SuagrCRM Ruby gem) up and running. (The SugarCRM Ruby gem relies on ActiveSupport >= 3, and therefore isn't compatible with Rails 2.)

Note: the Rails app we create in this article has been posted on GitHub at https://github.com/davidsulc/dev_zone_basic_rails_app, and where possible each step links to the specific commit its related to. Feel free to fork and experiment!

Getting started
---------------

First, let's create a new Rails app (https://github.com/davidsulc/dev_zone_basic_rails_app/commit/adb726e1a5d908953511164343b8ed2b798c5be8):

    rails new rails_demo --skip-active-record

You'll note we won't be using ActiveRecord, since we'll be dealing with records in SugarCRM and not in a local database.

We'll be using the sugarcrm ruby gem for this Rails app, so let's add it to the Gemfile (https://github.com/davidsulc/dev_zone_basic_rails_app/commit/3531c432baece1a47e20734ad255a08ee863e00a):

    gem 'sugarcrm', :git => 'git://github.com/chicks/sugarcrm.git'

In this case, we're using the git repository so everytime we update our bundle (with, e.g., `bundle update sugarcrm`), we'll be able to use the new features/bug fixes that have been added to the gem. We could also simply use `gem 'sugarcrm'`, but then we'd only gain access to new functionality as gems get released.

Next step, install the bundle, so all the gems we need (in particular the sugarcrm gem we just specified) get added to our app:

    bundle install

(Codebase after bundle install: https://github.com/davidsulc/dev_zone_basic_rails_app/commit/8d328c8a552a7e6cce08b1d3e0a6e1beeded6da7)

Configuring the connection to SugarCRM
--------------------------------------

Now that we have our environment up to speed, let's move on to configuring our app to connect to SugarCRM. Type `rails g` in console to see all the rails generators that are available to you. You'll notice that there's a 'SugarCRM' section which displays the generators made available to rails by the sugarcrm gem. Let's use one to generate the configuration for connecting to the SugarCRM server (https://github.com/davidsulc/dev_zone_basic_rails_app/commit/d82cd87337eb95fe915595d18118dae3370222dc):

    rails g sugarcrm:config

The generator will create 2 new files: `config/initializers/sugarcrm.rb` and `config/sugarcrm.yml`. You can safely ignore the former, but you need to fill in all relevant information in the latter: the URL where you SugarCRM server is, your username and password, etc. This information will be required for each Rails environment (i.e. development, test, production), just like in the typical `database.yml`. (If you've been following along so far, but don't have a SugarCRM server handy to test your app with, I highly recommend the no-fuss stack installers: http://www.sugarcrm.com/crm/download/sugar-suite.html#installers)

Before we move on, let's make sure the connection between our app and SugarCRM is working correctly. Open the rails console with `rails console` and try this:

    SugarCRM::User.first.user_name # => 'admin'

If you can't figure out what the line above means, you might want to check out our introduction to the sugarcrm gem: ...

Creating a model
----------------

And now, on to the actual Rails app itself. We're going to create a simple portal that will allow us to manage SugarCRM accounts. For simplicity, and to avoid cluttering the example, the only attribute we'll work with is the account's 'website' attribute.

First of all, let's create a rails model in `app/models/account.rb` (https://github.com/davidsulc/dev_zone_basic_rails_app/commit/d00727134257a51a5ddd3e1ba4f7d0169a1fa435):

class Account < SugarCRM::Account
end

You'll notice that it inherits from `SugarCRM::Account` instead of the usual `ActiveRecord::Base`. Other than that, this model doesn't do much: all the heavy lifting is passed onto the gem.

Let's check the magic is actually taking place: fire up the rails console and try

    Account.first.name # => should output a company name

Now, edit the account model (`app/models/account.rb`) again to add the following below the class definition (https://github.com/davidsulc/dev_zone_basic_rails_app/commit/d00727134257a51a5ddd3e1ba4f7d0169a1fa435):

    SugarCRM::Account.class_eval do
      def self.model_name
        ActiveModel::Name.new(Account)
      end
    end

This makes the `SugarCRM::Account` class return the model name we're using in our rails app.

Using a scaffold
----------------

Now let's generate an 'accounts' scaffold (https://github.com/davidsulc/dev_zone_basic_rails_app/commit/af11a295a6ac84ddd20811310d3c55fc397bfee4):

    rails g scaffold account name:string website:string --orm ''

(We need to specifiy an ORM since we're not using Active Record, except we don't want one. Specifying '' will allow the scaffold to be generated without using an ORM, don't worry when rails complains about not finding that ORM...)

Trying it out
-------------

Let's launch the server (`rails server`) to see what we have after aa few minutes of work. As you'll see by navigating to http://localhost:3000/accounts we already have a basic account portal that's fully functional!

Gem documentation can be found at https://github.com/chicks/sugarcrm, and questions/problems regarding the gem should be reported there also for quicker resolution.