---
layout: post
title: "ActiveRecord Serializers From Scratch"
date: 2013-07-26 11:38
comments: true
categories: [Rails, Ruby, ActiveRecord]
---

The Medeo app passes data between the backend Rails code and client side Javascript as JSON objects in three common ways:

1. via AJAX requests
2. through websockets
3. in rendered html pages via `data` attributes

This often meant reusing serialization code in different parts of the app often in different contexts. For this reason none of the popular template based JSON DSLs ([RABL](https://github.com/nesquena/rabl), [jBuilder](https://github.com/rails/jbuilder)) really fit our use case. [`ActiveModel::Serializers`](https://github.com/rails-api/active_model_serializers) was much closer to what we wanted but I was put off by its complexity and "magic". Eventually we ended up rolling our own simple, elegant solution that is serving us nicely.

To explain how we came up with this solution and how it works lets start with a simple example: a blog application that allows commenting. Comments are written by users so the app may have some `ActiveRecord` models that look something like this:

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

In order to create comments using AJAX and render our UI on the client side we need to return a JSON response that is composed of the comment's attributes as well as its user's attributes. This can be done easily using the `as_json` method provided by Rails' `Serialization` module. If you take a look at the [documenation](http://api.rubyonrails.org/classes/ActiveModel/Serializers/JSON.html), you will see that it accepts an optional hash that can be used to shape the format of the returned JSON. The specific options that we care about are the following:

- `:only` to limit the attributes included in the json
- `:methods` to include the result of some method calls on the model
- `:include` to include associations

Using these options, we can get the JSON we want as follows:

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

This is the result we wanted but it is still not quite and ideal solution. The `as_json` method call is definately a bit hairy. Reusing our serialization code would involve copying all of its arguments around, which would be a nightmare if we ever needed to change things. The natural way to clean this up is to create a class to encapsulate the responsibility of the serializing comments:

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

Now we can serialize comments by instantiating an instance of our serializer class: `CommentSerializer.new(@comment).as_json`. This has the added benifit of enabling us to test our easily test our serialization code in isolation. I personally think testing this type of serialization code is extremely important as it enforces the interface that connects your rails app to your javascript code.

Eventually we are going to want to serialize some models other than comments and it would be nice to reuse this pattern in these instances. One way to do this is to move the generic serialization bits into an abstract base class:

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
		def includes   ; end
		def methods    ; end
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

So far our solution is confined only to attributes, methods and associations defined inside of the class we are serializing. Obviously it would be nice if we could lift this constraint in order to create more complex JSON without packing our model full of methods that don't belong in it. One requirement that I come up against often when building applications is to return resource URLs in JSON responses. To do this we can simply include the Rails URL helpers in our base class, and then use them to augment the return value of a call to `super` from our derived class:

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

If you were paying close attention, you may notice that the above changes actually broke something. Before it provided its own `as_json` method, our `CommentSerializer` was capable of serializing both individual comments as well as collections of comments. This is because Rails provides an implementation of `as_json` for both `Array`s and `ActiveRecord::Relation`s that essentially map the collections by calling `as_json` on every object in the collection. We can do the same the same thing, by adding a layer of indirection in our `as_json` method and creating another method for individual instances:


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

The above code decects if it is serializing an individual model or a collection by testing if the `serialized_object` responds to `to_ary` (ie whether or not it can be coerced into an array). If it is a collection, it returns an array generated by serializing every element in the collection.

That's the entire solution. I think its quite powerful. I have created [a gist](https://gist.github.com/matthewrobertson/6129035) with the full code for the `BaseSerializer` and the `CommentSerializer`. As always, I am interested to hear any opinions / suggestions you might have.