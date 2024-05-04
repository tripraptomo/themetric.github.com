---
layout: post
title: "Protect your Rails attributes, even the foreign keys"
tags: [Ruby on Rails, security, Brakeman]
---
{% include JB/setup %}

All Rails developers are familiar with attributes and most understand the way Rails has built-in methods of protecting and exposing those attributes for the user and the internal code itself.  Before going through the possible security vulnerabilities associated with unprotected attributes, let's first see how Ruby and Rails make attribute methods and permissions for an object.

{% highlight ruby %}
# User model
class User < ActiveRecord::Base
  # Ruby
  attr_writer :first_name
  attr_reader :first_name
  attr_accessor :last_name

  # Rails specific
  attr_protected :admin, :role, :account_id
  attr_accessible :age, :bio
end
{% endhighlight %}

Above we have a simple User class which inherits from ActiveRecord and thus has access to our Rails database.  The first line is ruby code that uses a class method **attr_writer** that creates a setter method for an attribute called first_name.  If we interacted with an instance of this User model, we could set the user's name like this: `@user.first_name = "Paul"`.  The next line is **attr_reader** ruby method that allows use to read or get the value of our user attribute that we just set.  Finally, the **attr_accessor** method wraps the two above setter and getter methods into one, creating methods in a class that allow us to both write and read to the attribute last_name.  In the Rails world, classes (models like a User) have attributes that are automatically defined with getter and setter methods based on the schema defined in the database.  Because Rails does these attribute definitions for use in the model after we define them in the databse schema, it must give us a way to guard and limit what attributes can be accessed.  **attr_accessible** specifies a whitelist of attributes that are allowed to be updated all in one swoop.  **attr_protected** defines the attributes that should not be allowed (blacklisted) to be updated en-masse.

### In a database-backed application, shouldn't all attributes be accessible?

Since we defined our database-backed attributes in the database table schema, it would certainly make sense that all of these attributes should be "accessible."  If attributes are stored in a database, they are most likey changed and updated over time, as they should be!  The catch is that Rails methods like `update_attributes` which can be called on an instance of a model (like @user) exposed what is commonly called a "Mass assignmeht attack."  Let's setup a scenario where mass assignment becomes highly problematic and exposes our application to a security vulnerability.

{% highlight ruby %}
# User Schema
create_table "users", :force => true do |t|
  t.string   "first_name"
  t.string   "last_name"
  t.string   "email"
  t.boolean  "admin"
end

# User Model
class User < ActiveRecord::Base
  # invisibly has all attribute getters & setters for above database columns
end

# User Controller
class UsersController < ApplicationController
  def update
    ...
     @user.update_attributes(params[:user])
    ...
  end
end
{% endhighlight %}

All we're missing above is the view, but let's say that it's just a simple form with three input fields for the user first name, last name, and email.  When the user submits this form to update their user profile, Rails passes the form submission (as a PUT) to the update method of the Users controller and we have access to all these user-submitted values through params, specifically the `params[:user]` variable.  If we want to save what the user submitted to their account, we use the method `update_attributes` on the user instance @user (current user or otherwise).  Rails now conveniently takes all the submitted user params stored as an Array and saves them to the model (and thus database), updating that user.  The problem is that even though our user form might only have had three input fields, the user view is editable on the client side and in fact anything could be posted to our website through an HTTP PUT request.  If the user wanted to become admin, the HTTP post could include `params["user"]["admin"] => "true"` could be included as well and Rails would update the user model accordingly, it can't tell that changing the `admin` attribute (which perhaps gives special permissions) is much more dangerous than changing a first or last name!  This is where **attr_accessible** and **attr_protected** come into play by letting Rails know what attributes can be updated en-masse from an unregulated user HTTP post and saved directly to the database and which must be updated internally by Rails code.

#### Foreign keys can be susceptible too!

Foreign keys are frequently used as database and model relations to perhaps like a User to an Account by giving the user an attribute `account_id` that maps that specific user to a unique Account id.  Be sure and define specifically what attributes you want your user to freely update and change.  Defining an attribute like `account_id` in the `attr_accessible` whitelist would allow a user to submit any numerical Account id they wanted, even if you don't specifically provide a form for them to do so (HTTP posts can come from a variety of sources).  Leaving foreign keys out of the whitelist makes them blacklisted variables by default and if a user submits an HTTP post trying to update that attribute, Rails will throw a security error alerting of a mass assignment vulnerability.  In many cases, foreign keys are meant to be user-editable, for example when moving items in an account between a user's account collection, the user will want to be able to change an item's `collection_id` and that's perfectly acceptible to be whitelisted, just make sure that the user is moving the item to a collection that they actually own!

#### Make sure your app is locked down with Brakeman

Finding a vulnerability like the one described above can be difficult as it's often the case that code will be altered to produce less errors.  The Rails mass assignment warning could be annoying when you're rapidly prototyping an app and it's easy to just whitelist an attribute to squash the mass assignment warning message.  Later on you might forget to go back and protect that attribute because it's not causing any problems, yet it's still exposed to malicious hackers and curious users!  The [Brakeman scanner](http://brakemanscanner.org) is a static-analysis Ruby on Rails security scanner with a built-in check that scans your code for attributes defined as accessible when they probably shouldn't.  It also makes sure you have the appropriate settings defined in Rails to make sure you're protected against other mass assignment attacks.

### How does Rails 4 handle it?

Everyone upgrading to Rails 4 from a Rails 3 app is probably well aware of the move away from `attr_accessible` definitions in the model.  Instead, Rails protects against mass assignment using [StrongParemeters](http://edgeapi.rubyonrails.org/classes/ActionController/StrongParameters.html) which permits certain attributes in the controller parameters instead of the model.  The definitions are similar in both cases.  Except now we're specifying which attributes we want to allow users to set from the frontend interface, before it hits the model.  A controller for a view or an API might want to allow an admin user to update the role of an account member from author to editor.  In StrongParameters we would have to permit the `roles` attribute to allow this admin user to update the model.  But we certainly wouldn't want any user to update their role.  This is where authorization is important.  We want to authorize certain actions based on who's performing them.  The best tool I've found for this is a gem called cancancan (previously CanCan from Ryan Bates, the Railscasts guy).  CanCanCan can be found [here](https://github.com/CanCanCommunity/cancancan).
