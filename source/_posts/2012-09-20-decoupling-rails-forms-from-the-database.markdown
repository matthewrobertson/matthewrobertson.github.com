---
layout: post
title: "How To Stop Using Nested Forms"
date: 2012-09-20 01:31
comments: true
categories: [Ruby, Rails, Nested Forms]
---

One issue that caused me a lot of pain on my first few rails projects was the natural coupling that developed between the database and the rest of my application. The “skinny controller, fat model” mantra has been the prevalent in the Rails community since the early days. The problem with this philosophy is that it is only 50% accurate. If you are building good object oriented software, you shouldn’t have a fat anything.<!-- more --> When all of your business logic is encased in ActiveRecord objects, there can be some unfortunate consequences: things become difficult to reason about, it is hard to test objects in isolation and changing the database schema is a painful process that demands updates to many parts of the application.

In my opinion, one of worst offending features of rails is the ability to build nested model forms with `fields_for` and `accepts_nested_attributes_for`, as doing so directly couples your view layer to your database schema. Lately, I have been using a very simple technique to prevent this problem that I call building aggregate models.

## An Example

Consider as an example, an application in which the `User` model `has_one` associated `Email`:

``` ruby
class User < ActiveRecord::Base
  attr_accessible :name, :password, :password_confirmation
  has_one :email
  accepts_nested_attributes_for :email

  # validations etc...
end

class Email < ActiveRecord::Base
  attr_accessible :address, :confirmed
  belongs_to :user

  # validations etc...
end
```

The rails way to handle this association is to build a nested model form via `fields_for`. 

``` erb
<%= form_for @user do |f| %>
  <%= f.text_field :name %>
  <%= f.fields_for @user.email do |p| %>
    <%= p.email_field :email %>
  <% end %>
  <%= f.password_field :password %>
  <%= f.password_field :password_confimation %>
<% end%>
```

The obvious problem with this is that it couples the view directly to the database structure. If we decided to make changes to the database schema later, the form will need to be updated. I also find that `accepts_nested_attributes_for` is awkward to test and the subtleties of the api are difficult to remember and work with (e.g. mass assignment errors, associated validations).

Another option that many rails developers might opt for in this situation is to de-normalize the database and smash the `emails` and `users` tables together into one. In this case I decided to keep `emails` as a separate entity because they are going to have their own attributes (eg `verified?`). I also anticipate a requirement that users will have many emails. While there is [a case](http://www.codinghorror.com/blog/2008/07/maybe-normalizing-isnt-normal.html) to be made against normalization in some situations, the fact that it makes your view layer simpler to code is not one of them.

## The Solution

The approach I have been using in these situations has been to create a class to accept the form data and translate it to the active record layer. In Rails 3.0 the API required by controllers and views was extracted into [a set of modules](http://yehudakatz.com/2010/01/10/activemodel-make-any-ruby-object-feel-like-activerecord/) that can be included as needed. This allows us to create an object that is guaranteed to jive with `form_for` (or any other form gem you may be using) that is completely decoupled from `ActiveRecord`. To handle the example above, we might end up with something like this:

``` ruby
class Profile
  include ActiveModel::Validations
  include ActiveModel::Conversion
  extend ActiveModel::Naming

  attr_reader :user

  delegate  :name, :name=, :password, :password=, :password_confirmation, 
            :password_confirmation=, :persisted?, :id, :to => :user, 
            :prefix => false, :allow_nil => false

  def initialize(user, email)
    @user = user
    @email = email
  end

  def email
    @email.address
  end

  def email=(email_addr)
    @email.address = email_addr
  end

  def attributes=(attributes)
    attributes.each { |k, v| self.send("#{k}=", v) }
  end

  def save!
    User.transaction do
      @user.save!
      @email.save!
    end
  end

end
```

``` ruby Usage in the controller
def create
  @profile = Profile.new(User.new, Email.new)
  @profile.attributes = params[:profile]
  if @profile.valid?
    @profile.save!
    redirect_to some_url, :notice => "Huzzah!"
  else
    render :action => :new
  end
end
```

By adding a thin layer of indirection, this pattern reduces the coupling between the view layer and database. There are a few other big wins that come with it as well:

- the form markup is now as simple as it would be for one model with no associations
- we can move business logic out of the `ActiveRecord` classes and allow them to focus on their persistence responsibility (e.g. hash and salt the password before passing assigning it to `User`)
- we can add a different set of validations at the profile level (e.g. that are only pertinent to new users for example the confirmation of password)

Obviously we could enhance the `Profile` class to make it feel more like an `ActiveRecord` object (e.g. define `update_attributes` or a static `find_by_user_id` method that initializes the model for existing records) but for simple cases there is no need.

As always, this pattern should be used sparingly. Resist the urge to optimize prematurely.