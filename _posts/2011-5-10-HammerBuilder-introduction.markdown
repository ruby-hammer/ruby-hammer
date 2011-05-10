---
layout: post
title: HammerBuilder Introduction
---

`HammerBuilder` is a xhtml5 builder written in Ruby. It does not introduce anything special, you just
use Ruby to get your xhtml. `HammerBuilder` has been written with two main objectives:
* Speed
* Rich API
* Extensibility

## My motivation

I needed a fast Ruby html/xhtml builder for my jet experimental framework `Hammer`.
* `Markaby` is slow
* `Tagz` is really slow
* `Wee::Brush` is built-in `Wee`. I did not want to dig it out and as far as i know it creates instances for each tag.
That would be slow - lots of garbage.
* Erector was the fastest but it has `Widgets` which I did not need and they were complicating things not making
easier. Also there is no way of extending easily particular tags with a method.

I decided to create my own builder.

## Usage

There are two builders: `HammerBuilder::Standard`, `HammerBuilder::Formated`.
First one builds xhtml on one line unindented. Second one formats the output.

> Originally I did two builders because I thought formated one would be slower, but it is not.
> Anyway, I left there both - could be useful.

Personally I thing commented examples are the best, so here you are.

### Getting builders

Creating new builder is expensive. There is a pool of builders implemented.

{% highlight ruby %}
HammerBuilder::Formated.new # => new builder
b = HammerBuilder::Formated.get # => new builder from pool if there is one, or a new one

#  when builder is not needed any more
if am_i_smart?
  b.release!; b = nil # resets builder and returns it to the pool 
else
  b = nil # or the reference can be lost and it gets garbage-collected
end
{% endhighlight %}

> **Don't** forget to lose the reference in the first case. It's yours responsibility not to use builder after you have
> released it.

### Building xhtml

Basic syntax is similar to other html/xhtml builders you may know. I borrowed from
[Markaby](http://markaby.rubyforge.org/),
[Erector](http://erector.rubyforge.org/) and from not well known framework
[Wee](http://www.ntecs.de/projects/wee/doc/rdoc/) and its `Wee::Brush`es.

{% highlight ruby %}
HammerBuilder::Formated.get.go_in do
  xhtml5!
  html do
    head { title 'my_page' }
    body do
      div :id => 'content' do
        p "my page's content", :class => 'centered'
      end
    end
  end
end.to_xhtml! # =>
{% endhighlight %}
{% highlight html %}
<?xml version="1.0" encoding="UTF-8"?><!DOCTYPE html>
<html xmlns="http://www.w3.org/1999/xhtml">
  <head>
    <title>my_page</title>
  </head>
  <body>
    <div id="content">
      <p class="centered">my page's content</p>
    </div>
  </body>
</html>
{% endhighlight %}

But builder has internal class representation of each tag same as `Wee::Brush`. When `#div` is called its instance
is returned, following calls are possible.

{% highlight ruby %}
div.rclass # HammerBuilder::Formated::Div
# rclass is original ruby method #class 

div.id :id # <div id="id"></div>
div.class ['left', 'border'] # <div class="left border"></div>
meta.http_equiv "content-type" # <meta http-equiv="content-type" />
a.onclick 'js' # <a onclick="js"></a>
{% endhighlight %}

### Tag's content

How to get content into double tag:

{% highlight ruby %}
div 'content' # <div>content</div>
div.content 'content' # <div>content</div>
div :content => 'content' # <div>content</div>
div { text 'content' } # <div>content</div>
div.with { text 'content' } # <div>content</div>
div(:id => :id) { text 'content' } # <div id="id">content</div>
div.id :id do 
  text 'content'
end # <div id="id">content</div>
{% endhighlight %}

### Classes

Any attribute method you call is immediately appended to output with exception of classes.

{% highlight ruby %}
div(:class => 'left').class('center') # <div class='left center'></div>
div(:id =< 1).id(2) # <div id="1" id="2"></div>
{% endhighlight %}


## Things you should know about HammerBuilder

`HammerBuilder` is implemented as streaming library. Anything what can be flushed to output is flushed.
There are no multiple instances for each tag.
Every tag of the same type share a same instance (unique within the instance of a builder).

{% highlight ruby %}
puts(HammerBuilder::Standard.get.go_in do
  puts div.object_id
  puts div.object_id
end.to_xhtml!)
# output:
# 10069200
# 10069200
# <div></div><div></div>
{% endhighlight %}

Something like this cannot be constructed.
{% highlight ruby %}
puts(HammerBuilder::Standard.get.go_in do
  a = div 'a'
  div 'b'
  a.class 'class'
end.to_xhtml!)
{% endhighlight %}

Respectively can be, but output will be `<div>a</div><div class="class">b</div>`, because when `#class` is called the
second div is being builded.

`HammerBuilder` creates what he can prior to rendering and uses heavily meta-programming, because of that instantiating
the very first instance of builder triggers some magic staff taking about a one second. Creating new builders of the
same class is than much faster and getting builder from a pool is instant.

## Helpers

If they are needed they can be mixed directly into Builder's instance

{% highlight ruby %}
Hammer::FormatedBuilder.new.go_in do
  extend ActionView::Helpers::NumberHelper
  div number_with_precision(Math::PI, :precision => 4)
end.to_xhtml # => <div>3.1416</div>
{% endhighlight %}

> Be careful when you are using this with `Pool`. Some instances may have helpers and some don't.

or new builder descendant can be made.

{% highlight ruby %}
class MyBuilder < Hammer::FormatedBuilder
  include ActionView::Helpers::NumberHelper
end

MyBuilder.new.go_in do
  div number_with_precision(Math::PI, :precision => 4)
end.to_xhtml # => <div>3.1416</div>
{% endhighlight %}

## Extensibility

{% highlight ruby %}
class MyBuilder < HammerBuilder::Formated

  # define new method to all tags
  extend_class :AbstractTag do
    def hide!
      self.class 'hidden'
    end
  end

  # add pseudo tag
  define_class :Component, :Div do
    class_eval <<-RUBYCODE, __FILE__, __LINE__
      def open(id, attributes = nil, &block)
        super(attributes, &nil).id(id).class('component')
        block ? with(&block) : self
      end
    RUBYCODE
  end
  define_tag :component

  # if the class is not needed same can be done this way
  def simple_component(id, attributes = nil, &block)
    div[id].attributes attributes, &block
  end
end

MyBuilder.get.go_in do
  div[:content].with do
    span[:secret].class('left').hide!
    component('component-1') do
      strong 'something'
    end
    simple_component 'component-1'
  end
end.to_xhtml!
{% endhighlight %}

result is

{% highlight html %}
<div id="content">
  <span id="secret" class="left hidden"></span>
  <div id="component-1" class="component">
    <strong>something</strong>
  </div>
  <div id="component-1"></div>
</div>
{% endhighlight %}

## How to use HammerBuilder

The idea is that any object intended to rendering will have methods which renders the object into builder.
There is a `HammerBuilder::Helper` for that purpose.

Supose we have some `User` class with some builder methods `#detail` and `#attribute` defined,

{% highlight ruby %}
class User < Struct.new(:name, :login, :email)
  include HammerBuilder::Helper

  builder :detail do |user|
    ul do
      user.attribute self, :name
      user.attribute self, :login
      user.attribute self, :email
    end
  end

  builder :attribute do |user, attribute|
    li do
      strong "#{attribute}: "
      text user.send(attribute)
    end
  end
end

puts(HammerBuilder::Formated.get.go_in do
  User.new("Peter", "peter", "peter@example.com").detail self
  p "builder methods are: #{User.builder_methods.join(',')}"
end.to_xhtml!)
{% endhighlight %}

then we get:
{% highlight html %}
<ul>
  <li>
    <strong>name: </strong>Peter
  </li>
  <li>
    <strong>login: </strong>peter
  </li>
  <li>
    <strong>email: </strong>peter@example.com
  </li>
</ul>
<p>builder methods are: detail,attribute</p>
{% endhighlight %}

## Benchmark

`HammerBuilder` has been designed to be fast. So is it?

### Synthetic

{% highlight text %}
                              user     system      total        real
render                    4.380000   0.000000   4.380000 (  4.394127)
render3                   4.990000   0.000000   4.990000 (  5.017267)
HammerBuilder::Standard   5.590000   0.000000   5.590000 (  5.929775)
HammerBuilder::Formated   5.520000   0.000000   5.520000 (  5.511297)
erubis                    7.340000   0.000000   7.340000 (  7.345410)
erubis-reuse              4.670000   0.000000   4.670000 (  4.666334)
fasterubis                7.700000   0.000000   7.700000 (  7.689792)
fasterubis-reuse          4.650000   0.000000   4.650000 (  4.648017)
tenjin                   11.810000   0.280000  12.090000 ( 12.084124)
tenjin-reuse              3.170000   0.010000   3.180000 (  3.183110)
erector                  12.100000   0.000000  12.100000 ( 12.103520)
markaby                  20.750000   0.030000  20.780000 ( 21.371292)
tagz                     73.200000   0.140000  73.340000 ( 73.306450)
{% endhighlight %}

### In Rails 3

{% highlight text %}
BenchTest#test_erubis_partials (3.34 sec warmup)
           wall_time: 3.56 sec
              memory: 0.00 KB
             objects: 0
             gc_runs: 15
             gc_time: 0.53 ms
BenchTest#test_erubis_single (552 ms warmup)
           wall_time: 544 ms
              memory: 0.00 KB
             objects: 0
             gc_runs: 4
             gc_time: 0.12 ms
BenchTest#test_hammer_builder (2.33 sec warmup)
           wall_time: 847 ms
              memory: 0.00 KB
             objects: 0
             gc_runs: 5
             gc_time: 0.17 ms
BenchTest#test_tenjin_partial (942 ms warmup)
           wall_time: 1.21 sec
              memory: 0.00 KB
             objects: 0
             gc_runs: 7
             gc_time: 0.25 ms
BenchTest#test_tenjin_single (531 ms warmup)
           wall_time: 532 ms
              memory: 0.00 KB
             objects: 0
             gc_runs: 6
             gc_time: 0.20 ms
{% endhighlight %}

### Conclusion

Template engines are slightly faster than `HammerBuilder` when template does not content a lot of inserting or partials.
On the other hand when partials are used, `HammerBuilder` beats template engines. There is no overhead for partials in
`HammerBuilder` compared to using partials in template engine. The difference is significant for `Erubis`, `Tenjin` is
not so bad i did not find any easy way to use `Tenjin` in Rails 3 (I did some hacking).

So which one is better? It dependent on how much your rendering is fragmented or dynamic.

> But I thing It is save to say, `HammerBuilder` is comparable in speed with templates or even better.
> There is a rich API on top of that, I am happy with the results. **What do you think?**

