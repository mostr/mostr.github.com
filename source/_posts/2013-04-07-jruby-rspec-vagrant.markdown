---
layout: post
title: Continuous, fast Rspec tests with JRuby on Vagrant
date: 2013-04-07 11:12
comments: true
categories: jruby rspec test tools vagrant
---


I don't code in ruby on daily basis but from time to time I jump into ruby world. Most of the times I have my ruby playground on dedicated [Vagrant](http://www.vagrantup.com) virtual machine, so I can play with different setups, gems, packages and even if I break something it's just a matter of few Vagrant commands to get my environment back, in stable state. I use JRuby, but playing with JRuby has few well known drawbacks, one of which is startup time. It is even more important when continuous testing, TDD and quick feedback loop is what you believe in and use it all the day while coding. I prefer RSpec as my testing tool of choice in ruby world and there are few solutions to run tests on every change in files. One of them is [Guard](https://github.com/guard/guard). In its RSpec incarnation it runs `rspec` command every time file is changed. It works excellent on MRI, but is terribly slow on JRuby due to the fact every run of `rspec` command requires new java process to start (JVM needs to load all its stuff). There is a solution for that (see note at the end of this post), but the core thing with Guard is that it doesn`t play well with Vagrant.


When using Vagrant as a development environment I often have a project folder shared between my localhost and Vagrant VM so that I can write code locally using e.g. SublimeText and run this code as well as its tests on Vagrant machine. The issue with Guard is that it doesn't handle the case when file is being changed remotely. So it looks Guard isn't well suited for setups like mine. Looking for a solution I found another tool called [Watchr](https://github.com/mynyml/watchr). This is quite old gem with no active development (as far as I can see on Github), but does what I need - reacts on remote file changes. The simplest Watchr config to run RSpec may look as follows (for typical ruby project structure with `lib` and `spec` folders):


``` ruby .watchr
def run_spec(file)
  puts "Running specs: #{file}"
  `rspec -c #{file}`
end


watch("spec/.*/*_spec.rb") do |match|
  run_spec match[0]
end

watch("app/(.*/.*).rb") do |match|
  run_spec %{spec/#{match[1]}_spec.rb}
end
```

What it does is: it runs `rspec` on given spec file when either spec file or corresponding lib file was changed. To run it type `watchr .watchr`. It detects every file change even the remote one but still has this one huge drawback - runs `rspec` command every time which is terribly slow (JVM startup, ya remember?). It turns out that separate ruby doesn't need to be run every time to run RSpec. RSpec can be run from within existing ruby runtime via `RSpec::Core::Runner.run([file], STDERR, STDOUT)`. So it means that if Watchr runs its own JRuby runtime to watch for changes, this runtime can be reused to fire up RSpec internally. Let's change Watchr config file definition to look as follows:

``` ruby .watchr
require './watchr_spec_runner'

watch("spec/(.*/*)_spec.rb") do |match|
  WatchrSpecRunner.new.for_spec_file match[0]
end

watch("lib/(.*/.*).rb") do |match|
  WatchrSpecRunner.new.for_lib_file match[1]
end
```

where `watchr_spec_runner` is my tiny class as below:

``` ruby watchr_spec_runner.rb
class WatchrSpecRunner

  def for_spec_file(file)
    run_spec(file)
  end

  def for_lib_file(file)
    load_lib_file(file)
    run_spec(to_spec(file))
  end

  private

  def initialize
    @spec_dir = 'spec'
    @lib_dir = 'lib'
    @specs_suffix = '_spec.rb'
  end

  def to_spec(file)
    "#{@spec_dir}/#{file}#{@specs_suffix}"
  end

  def load_lib_file(file)
    load "#{@lib_dir}/#{file}.rb"
  end

  def run_spec(file)
    require 'rspec'
    RSpec.configure do |config|
      config.color_enabled = true
      config.tty = true
    end
    begin
      ::RSpec::Core::Runner.run([file], STDERR, STDOUT)
    rescue Exception => e
      STDERR.puts e.message
      STDERR.puts "Backtrace:\n\t#{e.backtrace.join("\n\t")}"
    end
  end

end

```




Now Watchr startup takes some time (as usual), but then running specs is blazingly fast. Of course provided Watchr configuration together with my `WatchrSpecRunner` class are dead simple and have just very, very basic funcionality - run spec when file changes. There is a lot more to do e.g. run all specs when going back to green from red phase, etc.

So this is what I came up with when talking about JRuby Vagrant and continuous RSpec run. For complete local development with JRuby I'd probably use Guard as it is more mature, has more features, is actively developed and easier to use.

**Guard with RSpec for JRuby note:**

There is an implementation of Guard-rspec for JRuby [here](https://github.com/jkutner/guard-jruby-rspec), but currently available version has an quite annoying issue - when you change spec file it "magically" doubles specs number run. [My pull-request](https://github.com/jkutner/guard-jruby-rspec/pull/22) with fix for that was already merged, so you may want to use latest version from Github until it is released.

