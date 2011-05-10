---
layout: post
title: "ActiveRecord: multiple databases"
---

I and a colleague of mine needed access to multiple databases with same model a few weeks back.
ActiveRecord sets database on class level, so we needed to generate classes for each database dynamically.
Here is the solution.

{% highlight ruby %}
class Remote < ActiveRecord::Base
  @abstract_class = true

  def self.db_context(db_name, &block)
    if block_given?
      block.call remote_class(db_name)
    else
      return remote_class(db_name)
    end
  end

  private

  def self.remote_classes
    @remote_classes ||= {}
  end

  # returns proper class from cache or creates new one
  def self.remote_class(db_name)
    remote_classes[db_name] ||= begin
      # creates dynamically descendant Class of self
      remote_class = Class.new(self)
      remote_class.class_eval { establish_connection db_hash(db_name) }
      remote_class
    end
  end

  def self.db_hash(db_name)
    # returns db hash for connection to db_name
  end

end

class RemoteSomething < Remote
  @abstract_class = true
  set_table_name "somethings"

  # actual model definition
  # validations, callbacks etc for Something
end

## usage ##

# finds all objects in DB db_name
RemoteSomething.db_context(db_name).all
# or with block
RemoteSomething.db_context(db_name) do |klass|
  klass.all
  klass.first
end
{% endhighlight %}

This is probably bad idea if number of remote databases is not limited or they change a lot, but this wasn't our case.
Source code is at <a href="http://gist.github.com/380518">http://gist.github.com/380518</a>.