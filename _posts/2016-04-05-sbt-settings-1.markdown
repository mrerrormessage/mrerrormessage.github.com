---
layout: post
title: Where do I put my settings? `sbt` as a configuration engine and manager
tags: scala, sbt
---

In my [prior post](http://mrerrormessage.github.io/2016/04/04/sbt-anatomy.html) we discussed the basic layout of an sbt project and how to get it to build a sort of hello world app.
In this post we're going to interact with sbt and use it to add the first major feature to our CLI app, listing all of the PRs in our Github repository.
So let's dive in and have a look.

## Libraries

To really get our project to do anything, we're going to want to add some libraries.
Since we want to write a maintainable project, first we're going to add a testing framework.
Once we have a very simple test up and running, we'll go ahead and add our feature

### Testing

Before we do anything too major, we're going to want to be able to run some basic tests.
When I programmed primarily in Ruby, absolutely *everything* had to be tested.
In Scala, the type system eliminates certain classes of errors (provided you use it responsibly), so you tend to only need to write tests around the actual behavior.
I tend to think that TDD is the way to go, and I'll be using it here to help drive the behavior of the project.
For testing, I'll be using the excellent Scalatest framework.
Specs2 and Scalacheck are also excellent frameworks as well.
I've used uTest in the past and it's also quite good, though more lightweight than the others.

We'll start by adding the dependency for the test framework to the `build.sbt` file.

#### Adding dependencies

```scala
//...

libraryDependencies += "org.scalatest" %% "scalatest" % "2.2.6" % "test"
```

If you'll recall the discussion of the `++=` operator from the last post, the `+=` operator is basically the same thing, but takes a single value instead of a sequence of values on the right-hand side.
The dependency is specified as a four strings.
The first string is the group ID, usually the reverse dns of the package creator.
It's followed by a `%%` which is another infix operator telling sbt to retrieve the appropriate version of the package for the version of scala used in this project.
The right hand side of `%%` is the artifact ID, simply the name of the package.
Since we used `%%`, sbt transforms the artifact ID by appending an underscore and the scala major version number to the artifact ID.
So `"org.scalatest" %% "scalatest" //...` is logically equivalent to `"org.scalatest" % "scalatest_2.11" //...`.
Following the artifact ID, we have the version, another `%`, and the scope.
The scope simply specifies what facet of the build this dependency is used in.
Since it's used in the `"test"` scope, we're saying that we need this when we're running tests, but not when we're running just the main module of the program.

#### Working with Settings

Then we'll close sbt and restart or simply type `reload` in the `sbt` console.
Note that `reload` basically does the same thing as stopping and restarting sbt, but without the need to close the JVM and start a new one.
Once sbt has restarted or reloaded, we'll take a minute to look at two commands that can be useful when working with `sbt`
The first is the `show` command, which displays the value assigned to a setting.

```
> show name
[info] gh-explorer
> show version
[info] 0.1-SNAPSHOT
> show libraryDependencies
[info] List(org.scala-lang:scala-library:2.11.8, org.scalatest:scalatest:2.2.6:test)
```

For any setting in sbt, we can query its value using `show`.
For those occasions when `show` doesn't give us quite all the information, we can use `inspect`.

```
> inspect name
[info] Setting: java.lang.String = gh-explorer
[info] Description:
[info]  Project name.
[info] Provided by:
[info]  {file:/home/rgrider/gh-explorer/}gh-explorer/*:name
[info] Defined at:
[info]  /home/rgrider/gh-explorer/build.sbt:3
// ...
> inspect libraryDependencies
[info] Setting: scala.collection.Seq[sbt.ModuleID] = List(org.scala-lang:scala-library:2.11.8, org.scalatest:scalatest:2.2.6:test)
[info] Description:
[info]  Declares managed dependencies.
[info] Provided by:
[info]  {file:/home/rgrider/gh-explorer/}gh-explorer/*:libraryDependencies
[info] Defined at:
[info]  (sbt.Classpaths) Defaults.scala:1259
[info]  /home/rgrider/gh-explorer/build.sbt:10
// ...
```

I've omitted parts of the output of `inspect` for the sake of brevity, but I just want to dig in a little bit with the information we can find out with inspect that will help us understand the build better.
On the first line, we have the type of the setting.
This will come in handy when we're working with defining settings based on other settings.
Next is a brief description, which is especially useful when working with cryptically-named or confusing settings.
Then we see "Provided by", which tells us where in the logical sbt hierarchy the setting falls.
Finally, we see "Defined at", which tells us which physical file(s) contain the setting information.

#### Writing and Running Our First Test

Now that we've added our dependencies and learned a bit more about how to work with settings in sbt, we'll go ahead and write our first test.
We'll put our test in `src/test/scala/ArgumentsSpec.scala`.
I'm going to write my tests in the [FunSpec style](http://www.scalatest.org/at_a_glance/FunSpec), but the excellent scalatest documentation shows how it can be written in any of scalatest's 8 styles.
So the test in `src/test/scala/ArgumentSpec.scala`:

```scala
package us.grider.ghexplorer                                                                                                                                                               
                                                                                                                                                                                           
import org.scalatest.FunSpec                                                                                                                                                               
                                                                                                                                                                                           
class ArgumentsSpec extends FunSpec {                                                                                                                                                      
  describe("parsing arguments") {                                                                                                                                                          
    it("should print help when no arguments provided") {                                                                                                                                   
      assert(Arguments.parse(Array()) == Help)                                                                                                                                             
    }                                                                                                                                                                                      
    it("should print help when -h argument provided") {                                                                                                                                    
      pending
    }                                                                                                                                                                                      
  }                                                                                                                                                                                        
} 
```

We can then flip over to sbt and run `test` and see that once we get our tests compiling, one of them is failing and the other is pending.

```
> test
[info] Compiling 1 Scala source to /home/rgrider/gh-explorer/target/scala-2.11/classes...
[info] ArgumentsSpec:
[info] parsing arguments
[info] - should print help when no arguments provided
[info] - should print help when -h argument provided (pending)
[info] Run completed in 177 milliseconds.
[info] Total number of tests run: 1
[info] Suites: completed 1, aborted 0
[info] Tests: succeeded 1, failed 0, canceled 0, ignored 0, pending 1
[info] All tests passed.
[success] Total time: 1 s, completed Apr 5, 2016 9:28:20 PM
```

If you want the full code, you can see the git repo (listing it here is really just clutter at this point).
Once we add the corresponding code, we'll see these tests pass nicely.
For the moment, we'll go ahead and simply stub out the list PRs functionality, which we'll come back and finish later.

### Recap

In this post we covered how to setup sbt to pull down maven dependencies, as well as what the fancy `%%` and `%` dependency builders actually mean.
We also looked at how to view our settings using `show` and `inspect`.
Finally, we wrote a test using the dependency and used the sbt `test` task to run it.
In our next post, we'll continue our in-depth look at settings in sbt by exploring:

* Settings which depend on other settings
* Dynamic environmental settings and when are settings evaluated
* Manipulating sbt settings using the sbt console
* Interacting with settings in the sbt project console

Stay tuned!
