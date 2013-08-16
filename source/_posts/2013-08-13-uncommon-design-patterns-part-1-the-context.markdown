---
layout: post
title: "Uncommon Design Patterns Part 1 - The Context"
description: "A look at Some Unconventional Problems"
date: 2013-08-13 19:51 
comments: true
categories: [ruby, patterns, object-oriented]
---

I'm planning to do a four-part series in the next few days (or weeks) illustrating some common design patterns that have come up at work and see a lot of use there.
These patterns are more used in some languages than others, but I have seen them used only very rarely in ruby code, which I think is a shame. Here is an example of the first one, the context.

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

## What is it?
This pattern executes a single use case.
We frequently use `#execute` instead of `#call` in our code, but the name is insignificant.
The job of this class ought to be to arrange and execute a given intention or procedure on the relevant set of domain objects.

## Why is it useful?
Because users want to perform a single action without knowing how.
We could teach users of our code to understand the domain model, and reach in and manipulate it to get what they want.
But then users might get it wrong!
Additionally, there is a lot of overhead in learning a domain model.
Once we invite users into the intricacies of our domain model, __woe to us__ when we want to change how it works even a little bit!
The context allows users to interact with our domain model without having to understand exactly how it works.

We strive to make the context self-encapsulated so that we can initialize once and call many times.
For instance, if I have five files that I want to change the owner of, it's nice to be able to go:
   
    file_changer = Directory::ChangePermissions.new(permissions_manager)
    [ 'a', 'b', 'c', 'd', 'e' ].each do |filename|
      file_changer.call(File.new(filename), 'me')
    end

## How did this pattern arise?
Use-case driven-design demands these objects and they are advocated by Bob Martin in his blog post [The Clean Architecture](http://blog.8thlight.com/uncle-bob/2012/08/13/the-clean-architecture.html). There's an excellent video on roughly the same topic from RubyMidwest. 
If you're interested in a little historical reading (albeit very dry), you might also have a look at Ivar Jacobson's "Object-Oriented Software Engineering: A Use-Case Driven Approach."

Another force driving adoption of this pattern within our team is the ease with which it allows us to test while injecting dependencies.
Instead of writing a test where we do something like

    describe Directory::ChangePermissions do
      let(:file) { double(:file) }
      before(:each) do
        Directory::PermissionsManager.stub(:has_write_permissions? => true)
      end

      it 'changes permissions if permitted by the permissions manager' do
        file.should_receive(:owner=).with(:owner)
        subject.call(file, :owner)
      end
    end

We can write a clean rspec test that doesn't require us to stub out methods on a class (always a win).

    describe Directory::ChangePermissions do
      let(:file) { double(:file) }
      let(:permissions_manager) { double(:file) }

      subject { described_class.new(permissions_manager) }

      it 'changes permissions if permitted by the permissions manager' do
        permissions_manager.stub(:has_write_permissions? => true)
        file.should_receive(:owner=).with(:owner)
        subject.call(file, :owner)
      end
    end

## Some Possible Objections

* Isn't this just imperative programming wrapped in a class?

No.

It's functional programming wrapped in a class.
What we have here is a closure over some dependencies.
This closure encapsulates certain behavior and data (like a class) and executes a certain action given an input.
This pattern when written in lisp is just

    (lambda (data) (dependency (... stuff here ...) data))

This lambda closes on `dependency` and executes on data.

* This isn't object-oriented at all - where is the domain model?

For the above examples you are totally right, there is no domain model whatever.
In more complex cases, a context instantiates, rehydrates, or queries the domain model, does something to it (a particular action to a particular member) and reports the result.
The reason you use a context here is because most people care much more about the result than about your domain model.

* But I can totally do this faster in a rails controller, can't I?

No, you can't.
Your tests will run slower because you will be testing rails and the database every time.
If you do this in a rails controller you are doing imperative programming in disguise.
There is some magic global-esque variable `params` that you inspect, do some work on singleton objects / classes (global variables) and you return some magical response.

* But I don't want everything in my app to know about this context's dependencies, how do I prevent that?

We'll discuss that soon. In the meantime, stay tuned for pattern 2, the repository.
