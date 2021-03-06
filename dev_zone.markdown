The Ruby gem for SugarCRM (https://github.com/chicks/sugarcrm) was already briefly presented to you in another Developer Zone article (http://developers.sugarcrm.com/wordpress/2010/11/16/web-services-in-your-own-language-part-2-ruby/), but has been drastically improved over the last months. In the following lines, we'll go over a few of these improvements and show you how leveraging SugarCRM can be greatly improved.

Setting up
----------

The gem is tested with Ruby 1.8.7 and 1.9.2, so you can pick your favorite version. Installing the gem is as easy s `gem install sugarcrm` (note: depending on your particular setup, you might need to call `sudo gem install` instead).

If you don't have a SugarCRM server setup to use for testing, try one of the stack installers (http://www.sugarcrm.com/crm/download/sugar-suite.html).

Since we'll be executing simple, nearly-independent commands, feel free to follow along in an irb session.

Connecting
----------

The first thing we need to do is `require 'sugarcrm'`, after which we can connect to the actual SugarCRM server:

    `SugarCRM.connect('http://127.0.0.1/sugarcrm','admin','letmein')`
    
You'll notice that the URL points to where the SugarCRM server is available on the web server, not the actual index page, nor the API endpoint.

If you're following along in irb, you'll see the gem dutifully returns a namespace similar to `SugarCRM::Namespace0`. You can safely ignore this for now (we'll tell you more about it in the advanced article).

Basic CRUD actions
------------------

On to our first subject: Create/Read/Update/Delete actions.

Let's start by seeing which contact was last entered in CRM:

    puts SugarCRM::Contact.last.last_name

All SugarCRM modules (i.e. "Contacts", "Opportunities", etc.) are namespaced within the `SugarCRM` Ruby module. So `SugarCRM::Contact` gives us access to the `Contact` Ruby class that will wrap all Contacts in CRM. (As a side note, any custom modules you may have created in Studio or loaded with the Module loader will be available in the same fashion.) From that class, we call the `last` finder function (more on those below) which will return the record in SugarCRM with the latest `date_entered` value. Finally, we call the `last_name` attribute on the contact instance to get the person's last name (which is then printed by Ruby).

Let's create a new contact:

    c = SugarCRM::Contact.create(:first_name => 'John', :last_name => 'Doe')

With respect to creating new class instances and records in CRM, the gem behaves much like ActiveRecord: `new` will create a new class instance without saving it to CRM, whereas `create` will actually save  the class instance on CRM (i.e. create a new record). As we called `create` in this case, our CRM should now have a new record. Let's check:

    puts SugarCRM::Contact.last.last_name # => Doe

Notice we're calling the same methods as above: we're still looking for the last created record. However, the output will be different, since we just created a new record. Let's change John Doe's last name:

    c.last_name = 'Dot'
    c.save!

To change the last name, we simply re-assign a value to the attribute. The attribute names always match the SugarCRM field names (which can be looked up in the Studio in the Admin interface), which means that any custom attributes (i.e. that have been added through the Studio tool) will end with `_c`. In other words, if we add a "middle name" field to our contact module using the Studio tool, we'll access that field in the gem by calling `my_contact.middle_name_c`.

Once again, the gem behaves like ActiveRecord: `save` will attempt to save the record. If it's unable to do so (e.g. some required fields are blank), it will simply return false. We'll call `save!` instead so that if the record can't be saved, an exception will be raised and the code execution will stop.

Now that we've gone through creating, reading, and updating, it's time to bid farewell to John Doe:

    c.delete

Associations
------------

Let's start by getting a SugarCRM account to work with. (We'll learn more about finding specific module instances later.)

    account = SugarCRM::Account.first

Let's see how many contacts it has:

    puts account.contacts.size

This account just hired a new receptionist. Let's start by creating a contact in CRM for her (just like we did above):

    alice = SugarCRM.create(:last_name => 'Angry', :first_name => 'Alice', :title => 'Receptionist')

Now we can add her to the account's contacts:

    account.contacts << alice
    account.save!

You can also call `account.associate!(alice)` which will link the two records and save the relationship in one step. There is a major implementation difference, though: when you call `account.contacts`, it loads all of the account's contacts before creating the new association, which might take a long time. If you just want to make sure the two records are linked and haven't already loaded the associated accounts, it's usually best to call the `associate` method.

Let's check Alice is now associated with the account, and see how many contacts this account now has:

    puts account.contacts.include? alice
    puts account.contacts.size

It turns out Alice wasn't a great fit for the company (she didn't really have people skills). Let's remove her from that account:

account.disassociate!(alice)

You could, of course, simply call `alice.delete` which would delete Alice's record in CRM and all associated relationships, whereas what we did above simply remove the link between the two records, without deleting Alice's record. (In other words, Alice is still in CRM.)

Let's print the full names of all contacts linked to this account, so we can make sure Alice isn't among them:

    account.contacts.each{|c|
      puts "#{c.first_name} #{c.last_name}"
    }

(Let's quickly remove Alice from CRM: `alice.delete`)

Dynamic finders
---------------

Just like ActiveRecord, the gem provides dynamic finders with attribute names. Here's an easy way to find the "Family Stationary" account:

    SugarCRM::Account.find_by_name("Family Stationary")

As you'll see below, this is simply a convenience method that calls a finder with a condition matching the attribute name (`name`, in this case) and value ("Family Stationary"). Such methods are available for all attributes defined on your CRM modules.

There is also an "all-in-one" method to ensure a record exists:

    SugarCRM::Account.find_or_create_by_name("Family Stationary")

If the record exsits, it will be returned. If no such record exists, it will first be created in CRM, and then returned.

Finders
-------

We've briefly seen how to use the default behavior of the `first` and `last` finders. As their name suggests, these finders will return, respectively, the first or last record matching the criteria (if no criteria is given, the default is to sort by date entered). But we can also give conditions to the query: let's search for the first account in Los Angeles.

  SugarCRM::Account.first(:conditions => {:billing_address_city => "Los Angeles"})

Notice that conditions are specified within a `:conditions` array. By default, the query will search for records where the attribute is equal to the specified value ("Los Angeles" in this case). But there are other ways to use conditons, as you will see below.

Let's find the first account with a zip code between 50000 and 800000. Notice how, instead of a single value, we've used an array to specify several conditions (including operators). These conditions will be joined with an SQL `AND` (if you want to use a query with complex boolean algebra, at the time of this writing you'll need to use the direct API method the gem provides, as explained in the "advanced" article).

    SugarCRM::Account.first(:conditions => {:billing_address_postalcode => ["> 50000", "<= 80000"]})

The last account in California that was created by the administrator (note how conditions are specified on multiple attributes):

    SugarCRM::Account.last(:conditions => {:billing_address_state => "CA", :created_by_name => "Administrator"})

The gem also has (very limited) SQL operator support, although they must be all uppercase to be recognized. Let's print all the accounts with "Inc" in their name:

    SugarCRM::Account.all(:conditions => {:name => "LIKE '%Inc%'"}).each{|a| puts a.name }

Notice how this time, instead of calling `first` or `last`, we called `all`. The finder syntax is the same for `all`, the difference lies in the return value: `first` and `last` will return either an instance matching the criteria or `nil`, whereas `all` will return an array of matching object instances (or an empty array if there are none).

Finders also accept some SQL arguments: let's print the list of the 10 last accounts (except the last one, due to the `offset` value) sorted descending by name:

    SugarCRM::Account.all(:order_by => "name DESC", :limit => 10, :offset => 1).each{|a| puts a.name }

Of course, you can also add a `:conditions` hash in there. Also to be noted: as of this writing the SugarCRM REST API seems to have a bug where the limit and offset options don't work correctly (the value for limit is always considered to be the smaller of the 2 values). This bug is fixed when using the gem: your query will return the expected results, even if the SugarCRM version you're using has this bug. That's how we roll...

Want to learn more? Check out other articles on this gem: how to create a basic Rails portal for SugarCRM using this gem, and advanced gem use (covering thins like using configuration files for automatic login, extending the gem, etc.).

Gem documentation can be found at https://github.com/chicks/sugarcrm, and questions/problems regarding the gem should be reported there also for quicker resolution.