---
layout: post
title: "Building a Pomodoro Timer in Dart - Part 1"
date: 2013-02-17 12:49
comments: true
categories: [dart, pomodoro]
---

Is Google Dart better than javascript? 
Google's Dart language has intrigued me for a long time. 
It fills the same role as javascript, but with an more traditional object-oriented bent and an (optional) strong typing system.

What I want to do over the course of several blog posts is to take the reader through a test-driven development of a dart pomodoro timer.
I may try to repeat this exercise in javascript in order to highlight the differences between the languages. 

So without further ado...

## The Dart Pomodoro Timer - Overview

The [pomodoro technique](http://www.pomodorotechnique.com/) is a system for sustaining mental effort through alternating work and breaks. 
It consists of working for a period of 25-30 minutes, then taking a break of 3-5 minutes. 
There are many pomodoro timers out there, but I'm writing this because it's a project that involves a touch of everything. 
There's a bit of system interaction (vis-a-vis clocks and timers), a bit of logic (how long is the pomodoro, how long is the break, where in the work-break-work-break cycle am I?), and a bit of UI (presenting the time to the user and having them start and stop). 
<!-- Need to use this paragraph to intro the whole blog post (probably linky-d) -->
But first, why don't we look at how to get Dart setup.

## Setting up Dart

To setup Dart visit [the dart language page](www.dartlang.org) and download the appropriate editor for your OS.
Notice you should get the editor package because it includes the SDK.
If you aren't a fan of IDEs, you don't need to bother with the Eclipse-lite Dart editor, any text editor will do.
Note that I am using Dart VM version: **Version Needed**. If you are using a later or earlier version you may get compiler warnings or errors.
Welcome to a language that is still under construction.

## Dart Timers and our Timer

Dart offers [Timers](http://api.dartlang.org/docs/releases/latest/dart_async/Timer.html) as part of the dart API.
As you can see, the timer is somewhat limited. It is initialized with a callback and continues to call that callback every so-many milliseconds until it is canceled.
I would like an abstraction over this, so I don't have to worry about the underlying details of the way a timer works.
I want a timer class that:

  - Takes some callback at initialization
  - Can be started arbitrarily
  - Can be stopped arbitrarily
  - Does not need to be given a callback when it is stopped and started 

Since what we are building will use a timer underneath, but is really about driving our application forward, we will call it a SecondTicker.
 
## Writing our SecondTicker

Given the above criteria, we specify our SecondTicker as follows:

  * The subject under test should invoke a callback once per second when started
  * The subject under test should not invoke a callback if not started
  * The subject under test should not invoke the callback multiple times per second if started multiple times
  * The subject under test should not invoke a callback while stopped

### Asynchronous Testing in Dart

Testing asynchronous code is notoriously hard.
A significant portion of the excellent [Growing Object-Oriented Software Guided by Tests](http://www.amazon.com/Growing-Object-Oriented-Software-Guided-Tests/dp/0321503627/ref=sr_1_1?ie=UTF8&qid=1362316939&sr=8-1&keywords=growing+object-oriented+software+guided+by+tests) (one of my favorite books on TDD) is spent treating the topic of asynchronous testing. Yes, it is _that_ hard. 
Fortunately, Dart has an amazing library for testing asynchronous code.
Dart's [scheduled_test library](http://api.dartlang.org/docs/releases/latest/scheduled_test.html) library handles the nitty-gritty of scheduling and running aynchronous tests for us, leaving us more time to focus on content and less time needed to focus threads and synchronizations.

### Testing SecondTicker

In order to test the first spec, we add the following code in second_ticker_tests.dart.

First, a few library imports: 

    import 'package:scheduled_test/scheduled_test.dart';
    import '../web/lib/pomo.dart';
    import 'dart:async';

Now we're off to the races. My biggest problem with dart testing is that it is done inside main. This makes it hard to separate tests out into separate files.
The strategy I've chosen is just to have mutltiple test files, all containing their own main. After main, we set up a few variables we will need in every test.

    main(){
      var count = 0;
      var counter = () => count++;
      var ticker = new SecondTicker(counter);
      ...
    }

Next we write the first test:

    test('The Second ticker invokes a callback once per second when started', (){
      //configure the test framework - we'll move this to setUp() in the future
      currentSchedule.timeout = new Duration(seconds:5);
      currentSchedule.onComplete.schedule((){ 
        return new Future.of(() => ticker.stop());
      });

      schedule((){
        return new Future.of(() => ticker.start());
      });

      schedule((){
        Completer completer = new Completer();
  
        new Timer(new Duration(seconds:1), (){
          expect(count, equals(1));
          completer.complete(true);
        });
        
        return completer.future;
      });
    });

This tests assertion is contained in the function passed to the test timer. 
It asserts that the second ticker has managed to count to one by the timer the Timer has counted to one.
Now to write the code that actually completes this task.

### Writing SecondTicker

Let's look at the absolute minimum of what this test requires. A callback to be called once.
And we have to have some stop and start methods.
Per our TDD training, we write the *absolute minimum* code required to make this test pass.

    class SecondTicker
    {   
      SecondTicker(callback){
        callback();
      }
      void start() => null;
      void stop() => null;
    }

Our tests pass:
    PASS: The Second ticker invokes a callback once per second when started

    All 1 tests passed.
    unittest-suite-success

### Adding Start

We create a proper setUp() method in our test to save effort later:
    setUp((){
      currentSchedule.timeout = new Duration(seconds:5);
      currentSchedule.onComplete.schedule((){ 
        return new Future.of(() => ticker.stop());
      });
    });

And we add a test for the new behavior we want to see:
    test('SecondTicker does not invoke the callback when it is not started', (){
      schedule((){
        Completer completer = new Completer();
  
        new Timer(new Duration(seconds:1), (){
          expect(count, equals(0));
          completer.complete(true);
        });
        
        return completer.future;
      });
    });

We see a failure as expected. We move on to make the test pass by writing as little code as possible.
We make our second ticker look like this:
    class SecondTicker
    {
      Function callback;
      SecondTicker(this.callback);
      void start(){
        callback();
      }
      void stop() => null;
    }

But we still see a failure! What's the deal? 

### The Pitfalls of Dart Testing
So Dart tests have some problems. As I mentioned above, they have to be run in main. They also have major isolation issues. Becuase they are run in main, we are leaking state between our two test cases. Fortunately, this is very easy to solve in this case:
    setUp((){
      count = 0;
      ...
    });

We now see the tests pass, as expected. On to the refactor step.

### Refactoring our tests
We've noticed some duplication in our test code. We have this block showing up in several places:
    schedule((){
      Completer completer = new Completer();

      new Timer(new Duration(seconds:1), (){
        expect(count, equals(0));
        completer.complete(true);
      });

      return completer.future;
    });

It's quite long, but really it is an assertion that we should have a certain count after a set duration. 
We're going to pull this out into a function.

    inOneSecond(Function assertion)
    {
      schedule((){
        Completer completer = new Completer();
    
        new Timer(new Duration(seconds:1), (){
          assertion();
          completer.complete(true);
        });
    
        return completer.future;
      });
    }

We rewrite our tests and notice a considerable improvement in length and readability.

### Working With the Timer
Finally the test that lets us see the Dart Timer class in action.
    test('SecondTicker should not invoke the callback multiple times when started multiple times',(){
      schedule((){
        return new Future.of((){
          ticker.start();
          ticker.start();
        });    
      });
      
      inOneSecond(() => expect(count, equals(1)));
    });

Writing the code to make this pass involves a bit more sophistication. We open up the docs on the [Dart Timer class](http://api.dartlang.org/docs/bleeding_edge/dart_async/Timer.html). The timer takes a callback, and can be canceled. This looks just a bit similar to ours. We try the simplest possible implementation.
    void start() {
      new Timer(new Duration(seconds:1), callback);
    }
We find that we *still* receive two callbacks. The callback is assigned to one timer on the first call to start and a different, separate timer on the second call to start. So it receives two callbacks in the space of one second. The author's first try at this yields another test leak.

We settle on this design:
    class SecondTicker
    {
      Function callback;
      Timer timer;
      
      SecondTicker(this.callback);
      
      void start() {
        if (timer == null) {
          timer = new Timer.repeating(new Duration(seconds:1), (t) {
            callback();
          });
        }
      }
      
      void stop() {
        if (timer != null) {
          timer.cancel();
          timer = null;
        }
      }
    }
We wanted to leave stop to a separate test, but we could not get the tests passing without if we didn't implement stop correctly here (stop is part of the cleanup code). In the end we have a simple, flexible design with 24 LOC.

##What's next?

We aren't done here. Stay tuned for part 2 of this series where we look at using mocks to build a new part of the pomodoro library.

