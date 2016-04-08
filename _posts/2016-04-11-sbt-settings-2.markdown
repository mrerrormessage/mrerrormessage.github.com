---
layout: post
title: Where do I put my settings? `sbt` as a configuration engine and manager (part 2)
tags: scala, sbt
---

In [our last post](http://mrerrormessage.github.io/2016/04/05/sbt-settings-1.html) we looked at adding a few settings to our project and using the `show` and `inspect` commands to look at what our settings are doing.
Today I want to add a few dependencies to the project and look at manipulating settings using sbt, as well as interacting with our project and libraries through the sbt console.
If you have the gh-explorer project downloaded, you can also follow along!

### Today's feature

Today we want to actually pull down and list all the PRs for a project in tabular form.
We're going to do this by interacting with the [Github Pull Requests API](https://developer.github.com/v3/pulls/#list-pull-requests).
To do that we're going to need an HTTP Library, a JSON library, and a little elbow grease, so let's get to work!

### Pulling down from Github

First we take a minute to make sure our CLI Argument Parser can pull out the name of the repository, as we'll need that to actually perform the action.
Then we'll turn our attention to the HTTP call to Github.
There are a number of excellent Scala HTTP libraries out there, I'm going to use [scalaj-http](https://github.com/scalaj/scalaj-http) as it's a simple, synchronous HTTP library.
If we were writing a server app we would almost certainly want an async library but a synchronous library should be just fine a for a basic CLI app.
Here comes a cool trick: we're going to add this dependency to our project without opening `build.sbt`!
In the sbt console, we do the following

```
> set libraryDependencies += "org.scalaj" %% "scalaj-http" % "2.3.0"
[info] Defining *:libraryDependencies
[info] The new value will be used by *:allDependencies, *:dependencyPositions
[info] Reapplying settings...
[info] Set current project to gh-explorer (in build file:/home/rgrider/gh-explorer/)
> session save
[info] Reapplying settings...
[info] Set current project to gh-explorer (in build file:/home/rgrider/gh-explorer/)
```

Now if we look in our `build.sbt` file, we'll see that the line adding the dependency has been added to the end.
How cool is that?

Anyhow, we want to do a quick test of the library just to make sure we can use it correctly with the Github API.
To do that, we'll run `console` in sbt.
`console` gives us an environment with our app and all its dependencies on the classpath, which makes it really easy to test out libraries or parts of your application.

```
> console
[info] Updating {file:/home/rgrider/gh-explorer/}gh-explorer...
[info] Resolving jline#jline;2.12.1 ...
[info] downloading https://repo1.maven.org/maven2/org/scalaj/scalaj-http_2.11/2.3.0/scalaj-http_2.11-2.3.0.jar ...
[info]  [SUCCESSFUL ] org.scalaj#scalaj-http_2.11;2.3.0!scalaj-http_2.11.jar (469ms)
[info] Done updating.
[info] Starting scala interpreter...
[info]
Welcome to Scala 2.11.8 (Java HotSpot(TM) 64-Bit Server VM, Java 1.8.0_60).
Type in expressions for evaluation. Or try :help.

scala> import scalaj.http.Http
import scalaj.http.Http

scala> val response = Http("https://api.github.com/repos/mrerrormessage/gh-explorer/pulls").asString
response: scalaj.http.HttpResponse[String] = HttpResponse([],200,Map(Access-Control-Allow-Origin -> Vector(*), Access-Control-Expose-Headers -> Vector(ETag, Link, X-GitHub-OTP, X-RateLimi
t-Limit, X-RateLimit-Remaining, X-RateLimit-Reset, X-OAuth-Scopes, X-Accepted-OAuth-Scopes, X-Poll-Interval), Cache-Control -> Vector(public, max-age=60, s-maxage=60), Content-Length -> V
ector(2), Content-Security-Policy -> Vector(default-src 'none'), Content-Type -> Vector(application/json; charset=utf-8), Date -> Vector(Fri, 08 Apr 2016 01:53:50 GMT), ETag -> Vector("80
b190627d4c87e9a37c34e20ea246a1"), Server -> Vector(GitHub.com), Status -> Vector(HTTP/1.1 200 OK, 200 OK), Strict-Transport-Security -> Vector(max-age=31536000; includeSubdomains; preload
), Vary -> Vector(Accept, Accept-Encoding), X-Con...
scala> response.body
res0: String = []

scala> response.code
res1: Int = 200
```

So we've now been able to test out the library and see that it plays nice with the Github API.
While there aren't any PRs for this repo, I tested it out on others with PRs and it works just fine pulling down PRs for those.
I'm writing a very minimal object with a single method `runCommand` to map from the `Command`s our application knows to requests for Github.
I'm not going to write a test for this class because it would be quite difficult to stub out the Github API and I have very little logic in the class.

### Parsing the json we receive

As part of getting the data we want, we're going to need to parse and transform the JSON response.
There are *many* JSON libraries for scala.
I'm going to be using json4s-native because (much like scalaj-http) it's a simple library that just works.
We'll go ahead and add this library to our dependencies:

```
> set libraryDependencies += "org.json4s" %% "json4s-native" % "3.3.0"
// ...
```

Now that we have the json library, we'll want to use it to parse the string we obtain using the http client.
We'll go ahead and write another test, fill it out and get the feature working.

### Wrapping up

We've learned a lot about settings and how to work with them in sbt.
Right now, we can list all the Pull Request IDs of Github Project, but there's so much more the Github API gives us.
Writing out a parser for all of the responses by hand would be tedious and error-prone, so we're going to generate it instead using sbt source generators.
Source generators use sbt tasks, so we'll get a good look at how those work as well.
Stay tuned for more scala and sbt fun!
