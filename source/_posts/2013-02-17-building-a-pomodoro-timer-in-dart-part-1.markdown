---
layout: post
title: "Building a Pomodoro Timer in Dart - Part 1"
date: 2013-02-17 12:49
comments: true
categories: [dart, pomodoro]
---

Google's new Dart language has intrigued me for a long time. It seems like javascript, but with a consistent semantics (less wat, less debugging) and an (optional) strong typing system. I really appreciate the ability to have a computer verify that my program will run correctly (or should). One of my favorite parts of working in languages like SML and Haskell is that the type system will catch most of the errors for you. Dart bucks the trend of languages like Ruby and Python that have more flexible (if less safe) type systems. However, because the typing is optional, you can still write code flexibly in early development, and add types later for safety. But I digress...

The last time I took a serious look at Dart was summer 2012. I was doing some small math/logic type programs in it to get a feel for the language and I ran into two main problems that put me off of it. First, the language was (and still is) in an evolutionary phase where there are frequent API-breaking changes in the language itself. Second, the tools were very immature. The dart editor crashed about once every 30 minutes on my 64-bit Linux mint based system. My experience with the tools was _much_ better this time around. I found the dart editor to be rock solid and was pleasantly surprised by some of its features. I see that the dart folks have an eclipse plugin in the works, but I was unable to get it working on my machine (64-bit Xubuntu 12.10, Eclipse 3.8, OpenJDK 1.7.0\_13).

What I want to do over the course of several blog posts is to take the reader through my development of a very simple dart pomodoro timer. I may repeat this exercise in javascript in order to highlight the differences between the languages, but I might not because I find javascript annoying for the reasons I listed above. I will note that while I do normally practice TDD I did not do so rigorously for this project both because (a) I am learning the language and wanted to experiment before writing tests, and (b) Dart's testing system still has some wrinkles. Nonetheless, I will present tests where possible. It ended up that most of my code in the final version was written in a TDD style because I had to go back and rewrite it after the first (non-TDDed) draft. 

So without further ado...

= The Dart Pomodoro Timer

The [pomodoro technique](http://www.pomodorotechnique.com/) for managing work and breaks. It consists of working for a period of 25-30 minutes, then taking a break of 3-5 minutes. There are many pomodoro timers out there, but I'm writing this because it's a project that involves a touch of everything. There's a bit of system interaction (vis-a-vis clocks and timers), a bit of logic (how long is the pomodoro, how long is the break, where in the work-break-work-break cycle am I?), and a bit of UI (presenting the time to the user and having them start and stop). 
