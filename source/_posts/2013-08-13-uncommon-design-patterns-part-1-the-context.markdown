---
layout: post
title: "Uncommon Ruby Patterns"
description: "A look at Some Unconventional Problems"
date: 2013-08-13 19:51 
comments: true
categories: [ruby, patterns, object-oriented]
---

At Sittercity I've recently begun to work with some developers who are newer to the team. 
Other members of the team and myself have developed a number of less-conventional patterns over the next month and when I show these patterns to the newer developers their form and function isn't immediately clear. 
I want to lay out three patterns that we use on our team that aren't frequently used in ruby, but have been very helpful to us.

Pattern One: The Directory

    module Directory
      attr_writer :permissions_manager

      def self.permissions_manager
        @dependency || PermissionsManager
      end

      def self.change_owner
        ChangeOwner.new(dependency)
      end
    end

What is it?
This pattern is a hand-rolled module specific DI container. It knows about how to build the objects in its domain and where to find their dependencies. 

Why is it useful?
Objects outside the domain no longer do not need to know about the construction of these objects, they just ask for one! It also allows for configuration and mocking in integration tests.

How did this pattern arise?
This pattern comes from working strictly within TDD.
Frequently we will have a test subject, say ThingOne and her tests will look like:

    descibe Directory::ChangeOwner do
      let(:permissions_manager) { double(:permissions_manager) }
      let(:file) { double(:file) }
      subject { described_class.new(dependency) }

      it 'changes the owner if the user has write permissions on the file' do
        dependency.stub(:has_write_permissions?).with(file).and_return(true)
        file.should_receive(:owner=).with('newowner')
        subject.execute(file, 'newowner')
      end
    end

While this is a stupid example, the point remains that it is really easy to test things when you inject them as dependencies.
As classes and systems become more complex, dependency injection helps to simplify things because I no longer ask "which thing do I do this to," instead I say "do that thing to the thing I'm holding."

Pattern Two: The Context

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
