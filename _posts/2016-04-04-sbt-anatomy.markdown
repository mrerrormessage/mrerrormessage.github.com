---
layout: post
title: The Anatomy of an Sbt project
tags: scala, sbt
---

## Intro

In today's post, I want to cover some basic Sbt conventions.
Like many systems today, Sbt uses convention over configuration in a number of different ways.
This is all good, as long as you understand and adhere to convention.

### Installing sbt

To run sbt you will need Java 6 or newer (I recommend Java 8).
You can find instructions to install `sbt` on [here on the Lightbend website](http://www.scala-sbt.org/release/docs/Setup.html).
Note that for the purposes of this project you can also use Activator (which is often easier to install).
If you cut out the Lightbend buzzwords, Activator is really just `sbt` with a nice web ui, templates, and a built-in editor.
I would **strongly** recommend Activator to people who are newer to programming or unfamiliar with the command line as there are a lot fewer things to configure.

### Starting up a project

To start with, I've created a new git repo for my project [gh-explorer](https://github.com/mrerrormessage/gh-explorer), for those following along.
This is going to be a simple scala toolbelt for various tasks I perform related to git and GitHub.
I'm going to walk through working on the project as a way to talk about the different features of sbt.
So starting off, we have an empty project, except for a license, a README, and a .gitignore.
Our goal by the end of this post is to have a barebones "Hello, World" type command application that simply runs, prints some text, and exits with 0.

### Files every project should have

The first file that every project will need is `project/build.properties`.
This tells `sbt` which version of `sbt` the build should use and (most importantly) allows you to use a single installed version of `sbt` with projects that use different versions of `sbt`.
This sorcery is accomplished in large part through the sbt launcher, which I hope to cover more in a later blog post.
So `project/build.properties` looks like this:

```properties
sbt.version=0.13.11
```

The second file we'll add is our `build.sbt` file.
You can think of `build.sbt` as the living heart of basically every sbt build.
It's like an ant `build.xml` or a maven `pom.xml` or a ruby `Gemfile` and `Rakefile` put together.
In other words, it's pretty awesome.
So what does it look like?
Well, a minimal one can be as short as:

```sbt
organization := "baz.qux"

name := "foobar"
```

This assumes a lot of defaults though, defaults that we don't necessarily want.
Instead, ours will look like the following:

```sbt
organization := "us.grider"

name := "gh-explorer"

scalaVersion := "2.11.8"

scalacOptions ++= 
  Seq("-deprecation", "-unchecked", "-feature", "-Xcheckinit", "-encoding", "us-ascii", "-Xlint", "-Xfatal-warnings")
```

At this point we should be able to start up `sbt` and get a look.
We'll see something like the following:

```
➜  gh-explorer git:(master) sbt
[info] Loading project definition from /home/rgrider/gh-explorer/project
[info] Set current project to gh-explorer (in build file:/home/rgrider/gh-explorer/)
> 
```

If we type `compile` at this point and hit enter, sbt will fetch the 2.11.8 compiler and library jars, and keep them ready for our use.

### What does the build.sbt mean?

So we've got a 4-line `build.sbt`, let's break down some of the meaning here.
Note that as this is an intro, it is *not* a complete explanation, but hopefully it gets us started.
It's pretty straightforward that the first three lines setup some basic properties about the build, even if we don't know exactly what `:=` means at this point (we'll get to that).
The final line is a bit more cryptic, as it introduces two constructs that we haven't seen before: `++=` and `Seq`.
In Scala `++` is a convention that almost always means something like "append two sequences".
Another Scala convention is that any operator followed by the equals sign means to apply the operator to the expression before and after and store the result in the variable before the operator.
Putting these two things together, we gather that the meaning is something like "take the sequence `scalacOptions`, add the sequence of strings starting with `"-deprecation"` to it, and store the result in `scalacOptions`".
This is, in fact, pretty much exactly what it means.
In a future post, I'll talk more in depth about settings and the different ways to access and transform them.

The reason that Scala conventions are used in sbt is because the `build.sbt` file *is* a Scala file.
You can do in it anything you would do in a normal Scala file, with a few exceptions.
This comes in handy when you want a build to be responsive to the environment, because you can have the `build.sbt` look up java system properties or environment variables to change the behavior of the build file.
So each operator above is translated to it's scala form.
Since in Scala the period and parens around an operator can be dropped, `name := "gh-explorer"` is translated to `name.:=("gh-explorer")`.

Where are `name`, `organization`, `scalaVersion`, and `scalacOptions` defined, you may ask?
Well, those are defined in the `sbt.Keys` Scala object, which `sbt` imports into the environment when compiling your `build.sbt`.
You can find a full listing [in the sbt.Keys scaladocs](http://www.scala-sbt.org/0.13.5/api/index.html#sbt.Keys$).
You'll notice that there are *a lot* of settings and our build file only touches 4 of them.
That's because in general the sbt defaults are great choices for building scala projects.
You'll notice that the type of most of the definitions in `sbt.Keys` scaladoc is `SettingKey[...]` or `TaskKey[...]`.
We'll dive into `TaskKey`s and `SettingKey`s more in a future post, but for now suffice it to say that the `:=` operator comes from the `SettingKey` type.
The reason we use `:=` instead of `=` (like we would with a Scala `val` or `var`) is because we aren't actually setting or changing the value referenced by the `SettingKey` or `TaskKey`, we're establishing a binding from the `SettingKey`/`TaskKey` to the value we specify.

### Compiling at last

So before we go, we should actually write some Scala today.
We'll create a new scala file at `src/main/scala/CommandLine.scala` that looks like this:

```scala
package us.grider.ghexplorer                                                                                                                                                               
                                                                                                                                                                                           
object CommandLine extends App {                                                                                                                                                           
  Console.println("-h: print this message and exit")                                                                                                                                       
}
```

We exit our text editor, re-open sbt and run the `compile` task.
`sbt` will think for a few seconds, then compile our file.
We can run our file as well, using the `run` task.
We see the following result:

```
> run
[info] Running us.grider.ghexplorer.CommandLine 
-h: print this message and exit
[success] Total time: 0 s, completed Apr 4, 2016 9:15:39 PM
```

So we've accomplished our goal of getting a basic project up and running.
Stay tuned for a more in-depth look at `sbt` settings as we add our first dependency and feature to the gh-explorer command-line app.
