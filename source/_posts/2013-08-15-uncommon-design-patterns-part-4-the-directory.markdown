---
layout: post
title: "uncommon design patterns part 4 - the directory"
date: 2013-08-15 20:57
comments: true
categories: [ruby, patterns, object-oriented]
---

Pattern Four: The Directory

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
