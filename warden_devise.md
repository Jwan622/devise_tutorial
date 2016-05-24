## Devise and Warden

The Devise gem is built on top of Warden. Warden is a Rack application, which means that it runs as a separate and standalone module, and is (nearly always) executed before the chief Rails application is invoked.

Warden provides the cookie handling that verifies the identity of a logged in user via a (secure) session string, in which the id (primary key) of a particular user is somehow stored and disguised. Warden also provides a hook so your app can deal with users who aren’t currently logged in. These users will either have restricted access, or none at all, except, of course, to sign-in/sign-up pages.

Warden knows nothing about the existence of your Rails app. As a result, Warden does not provide helper methods, controller classes, views, configuration options and log in failure handling. These things are what Devise supplies.

Devise often interacts with Warden using Strategies. (a strategy is a design pattern in which an algorithm is encapsulated within a dedicated class, which implements a method with a commonly shared name. You change the class to vary the algorithm). The Strategies that Devise employs are for encrypting passwords, email confirmations, and for HTTP Authentication. If you need to extend or augment Devise, you can implement a customized Strategy class. But this is an advanced subject, for which there is usually little call.


#### A Kick-Start Devise Installation

To have Devise completely installed in an existing Rails app, (where a user’s model class is named, User), the following six shell commands are required:

```rails
echo "gem 'devise'" >> Gemfile    # Or edit Gemfile and add line: gem 'devise'
bundle install                    # Fetch and install the gems
rails generate devise:install     # Creates config file, etc.
rails generate devise user        # Create model class, routes, etc.
rake db:migrate                   # Create user table
rails generate devise:views users # Creates (minimal) views
This assumes that the user model is called User.
```



#### Devise Configuration

Once you have installed Devise, as per the above commands, the next step is configuration. This is specified in two main files:

Firstly in a global config file, config/initializers/devise.rb. If you change the settings in this, they won’t become effective until you restart the Rails server. And the settings apply to the whole site.

Secondly, in a model class representing a (registered) user, which can be tailored to suit a particular role, for example if your app has an end-user, as well as having a site administrator (more on this later).

#### Devise Utility Methods

Devise contains dozens of classes, including, models, controllers, mailers, helpers, hooks, routes, and views, but since Devise exposes it’s functionality in a small number of simple helper methods, It’s unlikely that you will even need to know of the existence of all of them. The most important helper methods Devise gives you to use in your own app are:

```rails
authenticate_user!
current_user
user_signed_in?
sign_in(@user)
sign_out(@user)
user_session
```


#### authenticate_user!

The authenticate_user! class method (controller only), ensures a logged in user is available to all, or a specified set of controller actions. This method is invoked via a before_filter, for example:

```rails
# This class is intended to be a superclass from which
# the other (end-user related) controller classes inherit
class EndUserBaseController < ApplicationController
  before_filter :authenticate_user!
end
```

If the user isn’t logged in Devise backs off, and, by default, redirects to its own sign-in page. If a user is actually logged in, then Devise, by default, redirects to the root route. The root route is the default route for an app. It’s defined in config/routes.rb with something like the following:

```rails
root to: 'landing#index'
```

The authenticate_user! method’s use is for applications that require a logged in user to access (a set of) their pages. In this case, it is very common to use the current_user method (see just below) as the first receiver in a method chain that accesses a database, for cases where a record’s owner is to be stored. For example:

class SentMessagesController < EndUserBaseController
  before_filter :authenticate_user!

  def index
    @sent_messages = current_user.sent_messages.all
  end
end
The before_filter :authenticate_user! line ensures that current_user is never nil, thus avoiding the fatal error: method called on nil object.


#### current_user

The current_user method , whose purpose is self-explanatory, simply returns the model class relating to the signed in user. It returns nil if a user has not, as yet, signed in.


#### user_signed_in?

The user_signed_in? query method, which really just checks if the current_user method returns a value that’s not nil.


#### sign_in(@user) and sign_out(@user)

The sign_in(@user) and the sign_out(@user) methods. These are useful to login (or logout) a (newly created) user.


#### user_session

The user_session method, which returns meta data on a logged in user.



#### Regarding current_user and user_signed_in?
`current_user` and `user_signed_in?` are helper methods. So you can, for example, have a message in the header of your app stating the name of the signed in user.

Note that if you refer to an admin (who has a dedicated Admin class), the access method becomes current_admin, not current_user.

The prefix, or sometime suffix, is the corresponding underscored class name of a user class. For example, a user class name of ‘EndUser’, when underscored, becomes ‘end_user’, so the logged in user would in this situation be current_end_user.


#### Routes

Devise also defines a whole host of routes, which are defined in config/routes.rb with a line like:

```rails
devise_for :users
```


#### Devise Modules

When you invoke the Devise generator, a model class (in `app/models*) is created for you to modify for your specific requirements.

This user model class is where many of the most important configuration options are specified. Perhaps the most important of these are the Devise modules to use. These modules provide enhanced security functionality. There are ten modules in all.

The modules are included thusly:

```rails
class User < ActiveRecord::Base
  devise :database_authenticatable, :registerable,
          :recoverable, :rememberable, :trackable, :validatable,
          :confirmable, :lockable, :timeoutable
end
```


Note that when Devise generates a class similar to this for you, it also generates a database migration, which includes the fields on which the functionality of these modules depend. This migration file is well commented and it specifies which fields relate to which modules. It should be modified if you want to trim down the number of fields attached to the model. If some modules are not required, the fields related to them can be removed.


Although the Devise README lists these options, we’ll explain them a little here as well:

```
database_authenticatable – ensures that a user enters a correct password, and it encrypts the said password, before saving it.
confirmable – ensures newly registered users confirm their accounts by clicking a link on a (Devise sent) mail they receive. This is a defence against bots creating fake, but valid, users.
recoverable – for handling forgotten passwords safely.
registerable – allows users to register themselves, and subsequently change their login credentials.
rememberable – for transparently logging on a user, based on a cookie. This is the functionality associated with the
Remember me? checkbox you often see on login forms. Since the Devise config file specifies a timeout period, this is often of limited use. And is a security risk if the user steps away from their browser, with others about.
trackable – stores login information i.e. the user’s IP address, login time, last login time, and the total number of logins. These values are generally for site admins to look at, when trying to trace unusual activity.
validatable – ensures that the given email/name and passwords conform to a particular format.
lockable – limits the number of login attempts allowed before further access to a particular account is prohibited. Access restrictions are lifted after a time span, or if a (legitimate) user requests that an email be sent that contains a link to unblock his/her account.
```

These modules are lazily loaded, so if you omit some, the corresponding code is ignored.



#### Protecting the Mass-Assignment of User Attributes

Naturally, all of the fields associated with a Devise user contain sensitive data, whose alteration could seriously compromise security. For this reason, it is probably a good idea never to allow your app to have pages (which are accessible to untrusted users) that update (or create) a user model. Only allow these attributes to be updated by Devise, where you know it’ll be done securely.

If an app’s requirements ordain that auxiliary user information be kept, such as an address or a phone number, these details are best kept on an entirely separate table which has a foreign key back to the (Devise generated) user table. This auxiliary table would have a one-to-one relationship with the Devise user table.

If you really must add fields to the (Devise generated) user model you must absolutely make sure that Devise’s fields cannot be changed. The means by which you achieve this is different for Rails 4 apps than it is for Rails 3 apps. In the former case, it’s with strong_parameters in the controller. In the latter case, it’s with the attr_accessible or attr_protected class methods in the model.


#### Testing

It would be counter-productive to test Devise itself. It already has a comprehensive suite of tests that you are able run yourself, should you doubt its efficacy. However, you ought to test the parts of your app which interface with Devise, to ensure that Devise is presented with correct data, and to ensure that Devise remains in general working order.

Also, your own app’s tests are very likely to depend on your having a logged-in user, on whose behalf, the various operations of controller, acceptance, integration and view tests, are performed.

You can programmatically create a devise user as follows:

```rails
User.create!(email: "me@home.com", password: "watching the telly")
```
