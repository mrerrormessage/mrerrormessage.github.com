---
layout: post
title: "Uncommon Ruby Patterns"
description: "A look at Some Unconventional Problems"
date: 2013-08-13 19:51 
comments: true
categories: [ruby, patterns, object-oriented]
---

I'm planning to do a four-part series in the next few days (or weeks) illustrating some common design patterns that have come up at work and see a lot of use there.
These patterns are more used in some languages than others, but I have seen them used only very rarely in ruby code, which I think is a shame.

Pattern One: The Context

    module Directory
      class ChangePermissions
        def initialize(permissions_manager)
          @permissions_manager = permissions_manager
        end

        def call(file, new_owner)
          file.owner = new_owner if @permissions_manager.has_write_permissions?(file)
        end
      end
    end

What is it?
This pattern executes a simple use case.
We frequently use `#execute` instead of `#call` in our code, but the name is insignificant.
The job of this class ought to be to arrange and execute a given intention or procedure on a given set of objects.
This class is the authoritative source of how a given thing gets done in a system.

Why is it useful?
Sometimes, code needs to, you know, __do things__. 
This type of object is sometimes called an interactor (or controller, but not in the MVC sense).
A context encapsulates a use case. Nothing more, nothing less.
We put in dependencies in the the initializer (where possible) because it's nice to have a context be re-usable.
For instance, if I have five files that I want to change the owner of, it's nice to be able to go:
   
    file_changer = Directory.change_file
    [ 'a', 'b', 'c', 'd', 'e' ].each do |filename|
      file_changer.call(File.new(filename), 'me')
    end

Otherwise you have to initialize the same context five times, and while it isn't terribly inconvenient, this way is a bit nicer.
Plus you can pass it around without worrying about mutable state.

How did this pattern a rise?
Use-case driven-design demands these objects and they are advocated by Bob Martin in his blog post [The Clean Architecture](http://blog.8thlight.com/uncle-bob/2012/08/13/the-clean-architecture.html). There's an excellent video on roughly the same topic from RubyMidwest. 
If you're interested in a little historical reading (albeit very dry), you might also have a look at Ivar Jacobson's "Object-Oriented Software Engineering: A Use-Case Driven Approach."

Pattern Three: The Repository

    module Directory
      class FileRepository
        def initialize(filesystem)
          @filesystem = filesystem
        end

        def update(file)
          @filesystem.write(file)
        end 
      end
    end

What is it?
A repository is all about persisting an object.
Persistence is a messy problem, but having the object to be persisted deal with it creates more problems than it is worth.
In an upcoming blog post I hope to talk about why I believe ActiveRecord is an anti-pattern.
In the meantime
