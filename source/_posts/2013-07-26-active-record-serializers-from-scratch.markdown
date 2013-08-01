---
layout: post
title: "ActiveRecord Serializers From Scratch"
date: 2013-07-26 11:38
comments: true
categories: [Rails, Ruby, ActiveRecord]
---

- DSLs and template based solutions are a pain in the ass
- ActiveRecord serializers gem is close to what I wanted but too much magic
- ActiveRecord actualy provides a very powerfull API for object serialization
- as_json => to_json => to_json needs to return a hash that will be serialized


To explain how we came to this solution lets start with a simple example: a blog application that allows commenting. Comments are written by users so the app may have some models that look something like this:

```ruby
class Comment < ActiveRecord::Base
	attr_accessible :body
	belongs_to :user

	def html_body
		"<p>#{body}</p>"
	end
end

class User < ActiveRecord::Base
	attr_accessible :name
	has_many :comments
end
```

In order to create comments using AJAX and render our UI on the client side we need to return a JSON response that is composed of the comment's attributes as well as its user's attributes. This can be done easily using the `as_json` method provided by Rails' `Serialization` module. If you take a look at the [documenation](http://api.rubyonrails.org/classes/ActiveModel/Serializers/JSON.html), you will see that it accepts an options hash. The specific options that we care about are the following:

- `:only` to limit the attributes included in the json
- `:methods` to include the result of some method calls on the model
- `:include` to include associations

So using these options, we can get the JSON we want as follows:

```ruby
>>> @comment.as_json :only => %w(id created_at), :methods => %w(html_body), :includes => { 'user' => { 'only' => %w(id name) } }
=> {
	"id"=>1,
	"html_body"=>"lorem ipsum dolor...",
	"created_at"=>"2013-07-26T10:38:47-07:00",
	"user"=>{
		"id"=>1,
		"name"=>"Matthew"
	}
}
```

This is the result we wanted but it is still not quite and ideal solution. `as_json` is definately a bit of a hairy method call. Reusing our serialization code would involve copying all those `as_json` arguments around, which would cause problems if we ever needed to change things. The natural way to clean this up is to create a class to encapsulate the responsibility of the serializing comments:

```ruby
class CommentSerializer
	attr_reader :comment

	def initialize(comment)
		@comment = comment
	end

	def as_json(options={})
		comment.as_json(:only => attributes, :methods => methods, :includes => includes).merge(options)
	end

	private

		def attributes
			%w(id created_at)
		end

		def includes
			{ 'user' => { 'only' => %w(id name) } }
		end

		def methods
			%w(html_body)
		end
end
```

Now we can serialize comments by instantiating an instance of our serializer class: `CommentSerializer.new(@comment).as_json`. This enables us allows us to test our serialization code easily. I personally think testing this type of serialization code is important as it enforces the interface that connects your rails app to your javascript code.

Eventually we are going to want to serialize some models other than comments and it would be nice to reuse this pattern in these instances so lets move the generic serialization bits into a base class:

```ruby
class BaseSerializer
	attr_reader :serialized_object

	def initialize(serialized_object)
		@serialized_object = serialized_object
	end

	def as_json(options={})
		serialized_object.as_json(:only => attributes, :methods => methods, :includes => includes).merge(options)
	end

	private

		def attributes ; end
		def includes ; end
		def methods ; end
end
```

Now we can create serializers by simply inheriting from the `BaseSerializer` class and overriding the `attribute`, `includes` and `methods` hook methods. Making use of this base class, our `CommentSerializer` ends up looking something like this:

```ruby
class CommentSerializer < BaseSerializer

	private

		def attributes
			%w(id created_at)
		end

		def includes
			{ 'user' => { 'only' => %w(id name) } }
		end

		def methods
			%w(html_body)
		end
end
```

So far our solution is confined only to attributes, methods and associations defined inside of the class we are serializing. This will create a

One requirement that I come up against often when building applications is including resource URLs in JSON responses. To do this we can simply include the Rails URL helpers in our base class, and then use them to augment the return value of a call to super from our derived class:

```ruby
class BaseSerializer
	include Rails.application.routes.url_helpers
	# everything the same...
end

class CommentSerializer < BaseSerializer
	# everything the same...

	def as_json(options={})
		super(options).tap do |c|
			c['link'] = user_path(serialized_object)
		end
	end
end
```

If you were paying close attention, you may notice that the above changes actually broke something. Before it provided its own `as_json` method, our `CommentSerializer` was capable of serializing both individual comments as well as collections of comments. This is because Rails provides an implementation of `as_json` for both `Arrays` and `ActiveRecord::Relation` that essentially map the collections by calling `as_json` on every object in the collection. We can do the same the same thing, moving adding a layer of indirection in our `as_json` method and creating another method for individual instances:


```ruby
class BaseSerializer
	# everything the same...

	def as_json(options={})
		if serialized_object.respond_to?(:to_ary)
			serialized_object.map { |object| serialize(object, options) }
		else
			serialize(serialized_object, options)
		end
	end

	def serialize(object, options={})
		object.as_json({:only => attributes, :methods => methods, :includes => includes}.merge(options))
	end
end
```

The above code decects if it serializing an individual model or a collection by testing if the `serialized_object` responds to `to_ary` (ie whether or not it can be coerced into an array). If it is a collection, it returns an array generated by serializing every element in the collection.

That's the entire solution. I think its quite powerful. I have created [a gist](https://gist.github.com/matthewrobertson/6129035) with the full code for the `BaseSerializer` and the `CommentSerializer`. As always, I am interested to hear your opions / suggestions regarding this approach.