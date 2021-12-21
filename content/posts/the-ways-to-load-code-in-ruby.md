---
layout: post
title: "The ways to load code in ruby"
permalink: the-ways-to-load-code-in-ruby
date: 2016-02-18
comments: true
description: Exploring loading mechanisms in Ruby
keywords: require require_relative autoload load
categories: ruby-programming load require

tags: ruby load-code load require
---

### 前言
本周开始浏览下[Discourse源码](https://github.com/discourse/discourse)，发现一个和我们经常组织rails代码的方式不一样的用法，即大量使用`require_dependency`

经过一番查找和研究，发现`require_dependency`主要用于在developement环境下面，加载`./lib`目录下的class。类似这样的用法：
{% highlight ruby %}
# ./lib/meta_info.rb
class MetaInfo
  def name

  end
end
{% endhighlight %}

{% highlight ruby %}
# ./app/models/about.rb
require_dependency 'meta_info'
class About
end
{% endhighlight %}

主要目的在于当修改在`./lib`下的文件后，development环境下的会自动**reload**所有`./lib`下面的修改。`require`无法做到，必须重新启动rails server才行。

#### 深入研究Ruby中的代码加载机制
但是`require_dependency`仅仅是在rails中的一个load机制，所以不在这里过多介绍。但是由于好奇使用`require_dependency`和`require`之间的差异，所以完整的研究了下ruby下的load机制，他们分别是`load`, `require`, `require_relative`, `autoload`。例子使用User和Role之间的依赖关系来做比较，主要在User下如何加载Role。以下是Role的definition。
{% highlight ruby %}
class Role
  puts "loading role"
  def name
    "admin"
  end
end
{% endhighlight%}

{% highlight ruby%}
require 'forwardable' #ignore this require
class User
  puts "loading user"
  extend Forwardable
  attr_accessor :role
  def name

  end

  def_delegator :@role, :name, :role_name
end
{% endhighlight %}
如果在没有其它任何引用/加载机制下面，直接使用如下代码是会报错的。
{% highlight ruby %}
user = User.new
user.role = Role.new
user.role_name
{% endhighlight %}
{% highlight bash%}
NameError: uninitialized constant Role
{% endhighlight%}
因为在没有Role代码加载在程序运行时上下文的情况下，在User类是找不到Role的定义，使用ruby提供了一些代码加载的机制。
那么我们接下来看看这四种不同的机制下面，Ruby是如何加载代码的。

#### Kernel#load()
{% highlight ruby %}
# ./user.rb
load './role.rb'
{% endhighlight%}
而且在`irb`环境下面，使用load加载user.rb
{% highlight ruby %}
2.2.2 :001 > load './user.rb'
loading role
loading user
 => true
{% endhighlight %}
由于在代码中特别加入`puts ‘loading user’`和`puts 'loading role'`来测试代码加载中的执行机制，所以可以看到在使用`load`的情况下，user.rb和role.rb的所有代码是经过了一遍go through的，而且由于在user.rb第一行便执行`load './role'`，所以先print出”loading role“。

#### Kernel#require()
当把代码修改成这样
{% highlight ruby%}
# ./user.rb
require './role'
{% endhighlight %}
而且在`irb`环境下面，使用require加载user.rb
{% highlight ruby %}
2.2.2 :001 > require './user'
loading role
loading user
 => true
{% endhighlight %}
那么现在看来，似乎`load`与`require`没有什么区别，但是当我们执行运行第二次代码的时候会如何？
{% highlight ruby %}
2.2.2 :002 > require './user'
 => false
{% endhighlight %}
没有发生任何变化，这是说明在使用`require`的情况下，运行时上下文已经加载了代码，不会再额外加载，而`load`会完成再执行一遍。

#### Kernel#require_relative()
顾名思义`require_relative`是加载相对路径的代码，为了更好比较`require`和`require_relative`当把代码修改如：
{% highlight ruby%}
# ./user.rb
require_relative 'role'
# require 'role'
{% endhighlight %}
而且在`irb`环境下面，当执行require_relative，代码成功加载
{% highlight ruby %}
2.2.2 :001 > require './user'
loading role
loading user
 => true
{% endhighlight %}
但是当执行require的时候，在`require 'role'`的时候报错了。
{% highlight ruby %}
2.2.2 :001 > require './user'
LoadError: cannot load such file -- role
	from /Users/Paul/.rvm/rubies/ruby-2.2.2/lib/ruby/site_ruby/2.2.0/rubygems/core_ext/kernel_require.rb:54:in `require'
	from /Users/Paul/.rvm/rubies/ruby-2.2.2/lib/ruby/site_ruby/2.2.0/rubygems/core_ext/kernel_require.rb:54:in `require'
	from /Users/Paul/Programming/learning/workouts/require_vs_require_dependency/user.rb:2:in `<top (required)>'
	from /Users/Paul/.rvm/rubies/ruby-2.2.2/lib/ruby/site_ruby/2.2.0/rubygems/core_ext/kernel_require.rb:54:in `require'
	from /Users/Paul/.rvm/rubies/ruby-2.2.2/lib/ruby/site_ruby/2.2.0/rubygems/core_ext/kernel_require.rb:54:in `require'
	from (irb):1
	from /Users/Paul/.rvm/rubies/ruby-2.2.2/bin/irb:11:in `<main>'
{% endhighlight %}
找不到role，这是怎么回事呢？当我们加上`./`在role前，就是指定了一个相对路径给`require`。那么怎么解决这个Error呢？两个办法，当使用require保持显式路径，或者在user.rb第一行，加入
{% highlight ruby%}
$:.unshift '.'
{% endhighlight %}
这一行代码是讲加入当前路径加到`$LOAD_PATH`中，这样以来运行时上下文便有了当前路径，自然`require`就可以直接加载role了。`require_relative`则可以在当前文件的相对路径下加载其它代码。

#### Kernel#autoload()
保持争取的role加载在user.rb中，我们使用`autoload`的方式加载试试：
{% highlight ruby%}
2.2.2 :001 > autoload :User, './user'
 => nil
2.2.2 :002 > u = User.new
loading role
loading user
 => #<User:0x007f9cd9279700>
{% endhighlight %}
看到输出，可以理解为一种lazy load机制，它推迟了代码加载的阶段，直到最后代码需要调用的时候才会加载。看起来是一种性能较优的加载方式。
但是，当我们使用autoload的时候，如果文件路径错误，很难在没有调用的运行时检查错误。比如：
{% highlight ruby %}
2.2.2 :001 > autoload :User, './users'
 => nil
2.2.2 :002 > u = User.new
LoadError: cannot load such file -- ./users
	from (irb):2
	from /Users/Paul/.rvm/rubies/ruby-2.2.2/bin/irb:11:in ’<main>‘
{% endhighlight %}
代码加载时并没有提示任何错误信息，直到真正有方法的调用，才显示加载错误。

### 比较
- load需要每次执行代码，使用会带来更多的性能开销。
- require需要显式的执行相对文件路径，或者在require之前使用`$:.unshift '.'`，来加载当前文件路径到$LOAD_PATH中。
- require_relative为require提供一种更简单的相对路径使用方式。
- autoload在性能开销上有优势，但是对加载错误处理不如其他。

### 结论
- 包括很多gems都偏向于使用autoload，最有名的当属rails，个人认为在编程是能有效避免，或者在unit test阶段可以避免加载错误，autoload是一种不错的加载机制。
- 其次require，但是需要将文件显式的相对路径。
