---
---
# Rails and JRuby Caching <a name="rails-and-jruby-caching"/>

 


## Introduction
jruby-ehcache is a JRuby Ehcache library which makes a commonly used subset of Ehcache's API available to
JRuby. All of the strength of Ehcache is there, including BigMemory and the ability to cluster with Terracotta.
It can be used directly via its own API, or as a Rails caching provider.

## Installation
Ehcache JRuby integration is provided by the jruby-ehcache gem.  To install it simply execute
(note: you may need to use "sudo" to install gems on your system):

    jgem install jruby-ehcache

 If you also want Rails caching support, also install the correct gem for your Rails version:

    jgem install jruby-ehcache-rails2 # for Rails 2
    jgem install jruby-ehcache-rails3 # for Rails 3

## Configuring Ehcache
 Configuring Ehcache for JRuby is done using the same ehcache.xml file as
 used for native Java Ehcache.  The ehcache.xml file can be placed either
 in your CLASSPATH or, alternatively, can be placed in the same directory as
 the Ruby file in which you create the CacheManager object from your Ruby
 code. In a Rails application, the ehcache.xml file should reside in the
 config directory of the Rails application.

## Dependencies

*   JRuby 1.5 and higher
*   Rails 2 for the jruby-ehcache-rails2
*   Rails 3 for the jruby-ehcache-rails3
*   Ehcache 2.4.2 is the declared dependency, although any version of Ehcache will work

As usual these should all be installed with jgem.

## Using the jruby-ehcache API directly

### To make Ehcache available to JRuby

    require 'ehcache'

Note that, because jruby-ehcache is provided as a Ruby Gem, you must
make your Ruby interpreter aware of Ruby Gems in order to load it.  You
can do this by either including -rubygems on your jruby command line, or
you can make Ruby Gems available to JRuby globally by setting
the RUBYOPT environment variable as follows:

<pre><code>
 export RUBYOPT=rubygems
</code></pre>

### Creating a CacheManager
 To create a CacheManager, which you do once when the application starts:

<pre><code>
manager = Ehcache::CacheManager.new
</code></pre>

### Accessing an existing Cache
To access a cache called "sampleCache1":

<pre><code>
  cache = manager.cache("sampleCache1")
</code></pre>

### Creating a Cache
To create a new cache from the defaultCache

<pre><code>
  cache = manager.cache
</code></pre>

### Putting in a cache

<pre><code>
  cache.put("key", "value", {:ttl => 120})
</code></pre>

### Getting from a cache

<pre><code>
  cache.get("key")  # Returns the Ehcache Element object
  cache["key"]      # Returns the value of the element directly
</code></pre>

### Shutting down the CacheManager
This is only when you shut your application down.
It is only necessary to call this if the cache is `diskPersistent` or is clustered with Terracotta, but
it is always a good idea to do it.

<pre><code>
manager.shutdown
</code></pre>

## Complete Example

<pre><code>
class SimpleEhcache
 #Code here
 require 'ehcache'
 manager = Ehcache::CacheManager.new
 cache = manager.cache
 cache.put("answer", "42", {:ttl => 120})
 answer = cache.get("answer")
 puts "Answer: #{answer.value}"
 question = cache["question"] || 'unknown'
 puts "Question: #{question}"
 manager.shutdown
end
</code></pre>

As you can see from the example, you create a cache using CacheManager.new, and you can control the "time to live" value of a
cache entry using the :ttl option in cache.put.  Note that not all of the Ehcache API is currently exposed in the JRuby API,
but most of what you need is available and we plan to add a more complete API wrapper in the future.

## Using ehcache from within Rails

### The ehcache.xml file
Configuration of Ehcache is still done with the ehcache.xml file, but for Rails applications you must place this file in the
config directory of your Rails app.
Also note that you must use JRuby to execute your Rails application, as these gems utilize JRuby's Java integration
to call the Ehcache API.
With this configuration out of the way, you can now use the Ehcache API directly from your Rails controllers and/or models.
You could of course create a new Cache object everywhere you want to use it, but it is better to create a single instance
and make it globally accessible by creating the Cache object in your Rails environment.rb file.
For example, you could add the following lines to config/environment.rb:

<pre><code>
require 'ehcache'
EHCACHE = Ehcache::CacheManager.new.cache
</code></pre>

By doing so, you make the EHCACHE constant available to all Rails-managed objects in your application.  Using the Ehcache API is
now just like the above JRuby example.
If you are using Rails 3 then you have a better option at your disposal: the built-in Rails 3 caching API.
This API provides an abstraction layer for caching underneath which you can plug in any one of a number of caching providers.
The most common provider to date has been the memcached provider, but now you can also use the Ehcache provider.
Switching to the Ehcache provider requires only one line of code in your Rails environment file
(e.g. development.rb or production.rb):

<pre><code>
config.cache_store = :ehcache_store, {
                        :cache_name => 'rails_cache',
                        :ehcache_config => 'ehcache.xml'
                    "/>
</code></pre>

This configuration will cause the Rails.cache API to use Ehcache as its cache store.
The :cache_name and :ehcache_config are both optional parameters, the default values
for which are shown in the above example. The value of the :ehcache_config parameter
can be either an absolute path or a relative path, in which case it is interpreted
relative to the Rails app's config directory.
A very simple example of the Rails caching API is as follows:

<pre><code>
Rails.cache.write("answer", "42")
Rails.cache.read("answer")  # => '42'
</code></pre>

Using this API, your code can be agnostic about the underlying provider, or even switch providers based on the current environment
(e.g. memcached in development mode, Ehcache in production).
The write method also supports options in the form of a Hash passed as the final parameter.
The following options are supported:
* unlessExist, ifAbsent (boolean) - If true, use the putIfAbsent method
* elementEvictionData (ElementEvictionData)
* eternal (boolean)
* timeToIdle, tti (int)
* timeToLive, ttl, expiresIn (int)
* version (long)
These options are passed to the write method as Hash options using either camelCase or underscore notation,
as in the following example:

<pre><code>
Rails.cache.write('key', 'value', :time_to_idle => 60.seconds, :timeToLive => 600.seconds)
</code></pre>

### Turn on caching in your controllers
You can also configure Rails to use Ehcache for its automatic action caching
and fragment caching, which is the most common method for caching at the
controller level. To enable this, you must configure Rails to perform
controller caching, and then set Ehcache as the provider in the same way
as for the Rails cache API:

<pre><code>
config.action_controller.perform_caching = true
config.action_controller.cache_store = :ehcache_store
</code></pre>

## Sample Rails application
The easiest way to get started is to play with a simple sample app. We provide a simple Rails application
which stores an integer value in a cache along with increment and decrement operations.
The sample app shows you how to use Ehcache as a caching plugin and how to use it directly from the Rails
caching API. It is a simple demo application demonstrating the use of Ehcache in a Rails 3
environment.  This demo requires JRuby 1.5.0 or later.

### Checking it out

<pre><code>
svn checkout http://svn.terracotta.org/svn/forge/projects/ehcache-rails-demo/trunk ehcache-rails-demo
</code></pre>

### Dependencies
To start the demo, make sure you are using JRuby 1.5.0 or later.
The demo uses sqlite3 which needs to be installed on your OS (it is by default on Mac OS X).
There is a Gemfile which will pull down all of the required Ruby dependencies using Bundler.
From the ehcache-rails-demo directory:

<pre><code>
  jgem install bundler
  jruby -S bundle install
</code></pre>

### Starting the demo
You can start the demo application with the following command:

<pre><code>
jruby -S rails server -e production
</code></pre>

### Exploring the demo
To use the demo application, open a web browser to the following URL `http://localhost:3000/cache/index`.
This will display a simple screen allowing you to manipulate cached values
either through the Ehcache API directly, or through the Rails.cache API backed
by Ehcache.

## Leveraging the power of Ehcache
Once you have the Ruby/Rails caching modules up and running with Ehcache you can then
go on to leverage the power of Ehcache through for example creating a distributed cache
backed by Terracotta.
There are no limits on what you can do. Please see the rest of this documentation.