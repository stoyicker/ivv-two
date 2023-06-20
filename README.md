# stoyicker-ivv-two

## Building the project
This project uses Bazel as a buildsystem, and includes [Bazelisk](https://github.com/bazelbuild/bazelisk) binaries for Windows, Linux and Darwin. See [Installing Bazel](https://ij.bazel.build/docs/bazel-plugin.html#installing-the-plugin) if you prefer to invoke Bazel directly. You will also need JDK 7 or later. Then here are a few comamnds you can run:

```bash
bazelisk build //app:dev
bazelisk mobile-install //app:dev 
```

### Importing the project in Android Studio
1. [Install the Bazel plugin](https://ij.bazel.build/docs/bazel-plugin.html#installing-the-plugin).
2. Open the project by selecting [WORKSPACE](WORKSPACE) from File > Open Workspace File... 

The project includes run configurations so, once synced, you can install and run the app from the IDE interface if you do not want to deal with the terminal.
#### Known issues
```bash
Error:WORKSPACE:_:_://external:android/sdk depends on @androidsdk//:sdk in repository @androidsdk which failed to fetch. no such package '@androidsdk//': Either the path attribute of android_sdk_repository or the ANDROID_HOME environment variable must be set.
```

This is due to a 'problem' in the Bazel plugin for IntelliJ. Basically, you need to force the ANDROID_HOME env var into Android Studio. In Linux, you can do this by running Android Studio from a prompt where ANDROID_HOME is set, or setting it locally to the command by prefixing it:

```bash
$ ANDROID_HOME=<your_android_HOME> </path/to/studio.sh>
```

See [bazelbuild/intellij#102](https://github.com/bazelbuild/intellij#102) for more info and workarounds and [bazelbuild/bazel#4047](https://github.com/bazelbuild/bazel#4047) for how this should be addressed in the future.

### Why Bazel?
With the issue stated above, a [roadmap](https://github.com/bazelbuild/rules_android/blob/master/ROADMAP.md) that shows progress being notably behind schedule and a community pretty much non-existent when compared to other alternatives, why not use, for example, Gradle and the Android plugin? Reasoning follows:

* __Documentation__: [this](https://google.github.io/android-gradle-dsl/current/) is the class reference of the Gradle plugin; pretty much everything is a useful as the javadoc of a getter, and its only really relevant use case is to justified by the lack of autocomplete when using AGP via Groovy. The [Android documentation for the Android rules for Bazel is a first-class entry in the Bazel website](https://docs.bazel.build/versions/master/tutorial/android-app.html), in spite of them not having reached Alpha yet. The Android rules for Bazel also have a public reference on the Bazel site, much more complete than that of the Gradle plugin.

* __`bazel mobile-install`__: See [Problems with traditional app installation](https://docs.bazel.build/versions/master/be/android.html) and [The approach of bazel mobile install](https://docs.bazel.build/versions/master/mobile-install.html#the-approach-of-bazel-mobile-install).

* __Flexibility__: The Android plugin for Gradle is very strict with regards to how you are allowed to organize files. For example, it is very common to have an app with different flavors where some of those flavors want to offer particularities, for example particular implementations of a class. In these cases, the Android plugin for Gradle expects a version of that class in every flavor, so in the best of cases you are forced to adapt your architecture to work around this, use symlinks or custom scripts to fool the plugin, or in the worst one just having to maintain multiple copies of files containing functionality that is to be used by more than one flavor. Resources feature similar issues, where for example there is no built-in way to provide resources shared across flavors, and the only way to provide a set of fallback resources that is guaranteed to cover all common resource demands is to define a flavor specifically for this purpose. All of these issues can be worked around by making extensive use of Gradle modules. Like Gradle, Bazel does not have a concept of flavors, but neither do the Android rules for Bazel - and, in my opinion, correctly so. Instead, they are just a nice API to process the files developers write and pack them up into Android binaries, but implies nothing about how such files must be organized or related to each other.

* __Scalability__: While, as mentioned above, the shortcomings of the Gradle plugin for Android can be addressed via multi-module structures, these are a nightmare to handle. Experiences of ridiculous heap requirements and build times aside, Gradle mult-module projects aren't really "multi" - they require a special top-level project that acts as a glorified container for sub-projects, and there can only be one of these in a project, meaning that module nesting is impossible, which is really bad for scalability. In addition, this magic top-level project has the capability to influence sub-projects, with clauses such as for example `allProjects`. However, Gradle has this concept of [Decoupled projects](https://docs.gradle.org/current/userguide/multi_project_builds.html#sec:decoupled_projects) which explains how its capabilities to building multi-module projects within a sane amount of resources/time are hindered if you opt to use the top-level project for what you would normally want to use it: [configuration injection](https://docs.gradle.org/current/userguide/multi_project_builds.html#sec:cross_project_configuration). This paragraph is of particular interest:

  > A very common way for projects to be coupled is by using configuration injection. It may not be immediately apparent, but using key Gradle features like the allprojects and subprojects keywords automatically cause your projects to be coupled. This is because these keywords are used in a build.gradle file, which defines a project. Often this is a “root project” that does nothing more than define common configuration, but as far as Gradle is concerned this root project is still a fully-fledged project, and by using allprojects that project is effectively coupled to all other projects.

  In addition, there is a major problem with Gradle's build model: from any build file, plugins included, it is possible to hook up to any of the different phases of the build process and modify it or create dependencies to and, most importantly, from it. This means that, accidentally, you could cause another module, say Foo, to depend on the one you are working on without touching Foo at all. On top of all this, there is no way to systematically determine whether a project is decoupled or not, which can become a very serious problem with as projects get bigger and bigger.

  Bazel's version of Gradle modules is called packages. A package is a combination of targets, which is an umbrella term for files, rules and package groups. Package groups are very marginal and irrelevant in this repo, and files are obvious. A rule is somewhat equivalent to a Gradle task in that it is Bazel's way to process files. Rules can take as inputs files and/or other rules (even from other packages), and output files. Rules feature a [similar three-stepped lifecycle to Gradle tasks](https://docs.bazel.build/versions/master/skylark/concepts.html#evaluation-model), but with a very key difference: Bazel's top-level container is not a target, it's something different: a workspace. This is represented by a file where the only thing that can be done is configuration injection. Unlike its Gradle equivalent, Bazel workspaces are not related to their packages, and it is impossible to misuse them as a package because they are not packages. This means that the only ways for packages to be coupled to each other is via rules or direct dependencies. This is the correct way to do it in Gradle - from [Decoupled projects](https://docs.gradle.org/current/userguide/multi_project_builds.html#sec:decoupled_projects), *decoupled projects may only interact in terms of declared dependencies: project dependencies and/or task dependencies* - but it is enforced in Bazel. Finally, Bazel implicit transitive dependencies are represented in data structures called [depsets](https://docs.bazel.build/versions/master/skylark/depsets.html) that are immutable, which makes the case where a dependency could accidentally be added to a Gradle module that we are not working on impossible in Bazel.

## Architecture
This project is split in Bazel packages. There is not a hard, black-or-white criteria to split them because I have not been able to come up with one. Instead, I split in different packages things that I want to make sure not to accidentally couple to each other. This means that some packages only have UI, and some others have things that are unrelated to UI, as the UI/non-UI separation is something I consider very important to respect.

The project heavily relies on Dagger2 for dependency injection. Every instance of non-platform types is provided via Dagger to maximize development flexibility and testing capability. Packages containing an Application class use it to hold a root component, otherwise they defer this responsibility to a local ContentProvider that gets merged into the manifest when everything is put together.

The code in each package is split into different Java packages following the Dagger graph; a Java package exists for each (sub-)component and, if a dependency is injected by a certain (sub-component), the class of that dependency lives in the package of that component.

If instances of a class are provided from more than one component, there are two possibilities:
* If the components from which these instances are provided share an ancestor, directly or indirectly, and the class can be provided from the lowest common ancestor without exposing it to additional nodes that do not need it, provision of instances of that class is moved to said lowest common ancestor, the class definition is therefore moved to the package of said lowest common ancestor and scoping is controlled via `@Binds` in the subcomponents.
* If the components from which these instances are provided share an ancestor, directly or indirectly, and the class __cannot__ be provided from the lowest common ancestor without exposing it to additional nodes that do not need it, the class definition is moved to the package of salid lowest common ancestor but provision of instances is not altered.
* If the components from which these instances are provided do not share an ancestor, something's gone wrong and needs fixing because in application packages we should always have a single root component that hangs from our custom Application class and in library packages similar story but from a local ContentProvider instead.

While these packaging rules may not be common, they do not allow room for interpretation and enforce the file structure to closely resemble the architecture of the software, which favours simplicity, maintainability and readability.

Finally, bear in mind that while the name of some classes (e.g. use cases) may remind you of Clean Architecture, which is in my opinion a very correct concept, I don't aim to respect it fully because it offers things that I do not need (such as the capability to reuse a domain layer, for example), so take anything you find along these lines as simple evidence of the influence on my practices of using Clean Architecture repeated times over the years, and not as a half-way attempt at imitating it.

## FAQ

### Why no Kotlin?
I've tried Kotlin for several years, at work and on the side. You can filter my repositories by name to see how I've used Kotlin to build apps, libraries and even annotation processors. Over time, I have learned that, in my opinion, it's just not worth. Let's look at the trade-offs in detail:

* Requires extra build setup. For Gradle, this means an extra plugin, plus a second one for annotation processing, which is almost a given nowadays. This is less relevant in Bazel, but still.
* Requires adding the Kotlin stdlib to the classpath.
* Dagger2 injection of generic types requires `@JvmSuppressWildcards`, which is a bit of an annoyance when an app uses Dagger2 extensively (like this one).
* I don't like type inference. It encourages relying on specific types by default, that is, tempts to break [Liskov's substitution principle](https://en.wikipedia.org/wiki/Liskov_substitution_principle) principle, and, although in Java you can always downcast an object no matter 
how it is defined as, the fact that its type needs to be defined explicitly means that at least you're as close to using the lowest required abstraction (correct) as to using the lowest type (wrong).
* No semi-colons, `object`, trailing lambdas, `infix`, `tailrec`. Just candy, I don't see features like these justifying any sort of trade-off.
* Compiling to bytecode of old versions of Java means that the evolution of the available API is independent of whatever version of Java is supported by Android. This is real neat, but the same can easily be achieved by writing a library with a bunch of utility functions (this is essentially what Guava is, for example).
* Auto-generated value types via `data`. This is comfortable, but using AutoValue is the same thing and does not require anything else.
* Explicit `val`/`var` requirement. Java has `final`, but it being optional unless there are multiple threads available makes it so most people skip it. However, on a project where only I work, I don't need to worry about others misusing the available toolset.
* Multi-platform. Yes, great and all, but I'm just building an Android app, so this is irrelevant to me.
* Classes are final by default.
* `internal`.

Some of these cons could be looked at from the "very small issues" standpoint, but bear in mind I'm comparing it to just using Java, which is simpler to build, does not have these issues, so there is no reason to "forgive" small issues when there is the alternative of not having to deal with them at all when only giving up a couple of points (the last ones) that I consider actually valuable.

### Why no Android arch components/Room/LiveData/etc?
I'm lucky enough to have spent most of my time as an Android developer either in a time where these components did not exist or working on projects that could not afford treating the classpath like a rich child's wish list for Santa (libraries, embedded systems, etc). As such, I've been forced to 
learn how to do things with very little tooling, and now that I know how, comparing this approach with how these libraries force specific ways of working (for example, the fragment injection of the whole lifecycle-awareness 'suite') and try to reinvent the wheel with dubious results (compare 
LiveData vs RxJava, Room vs literally just writing SQLite manually) I immediately say no; there's no reason to add a bunch of dependencies to my code that limit me in how I can do things.

### Why gRPC?
I think the technicalities of gRPC make it very superior to the otherwise 'normal' option, REST. Even putting aside things like HTTP2, reduced transfer size or bi-directional streaming that don't have relevant much of a relevant impact in a client demo project, just the simplicity of taking the interface definition of a service and its Protocol Buffer models
and auto-generate all the code required it's just miles away of any other approaches available. Plus, I bet you've seen many apps using REST and JSON, but how many with gRPC?

Note also that I'm fully aware that for the particular API of this project I know that there are official libraries available which would hide the 'metal' a bit. But the purpose of this project is to showcase, so why hide cool stuff away?

### So are you not comfortable with Kotlin/REST/X popular library or pattern you're not using?
Most likely I absolutely am. I am not trying to tell you what I can't do - instead, this is an opportunity for me to what my way of thinking, applied to my experience, has deemed as the closest-to-ideal setup for an Android application project, specially for a showcase one. But, as can be expected, almost all the work I've done is with the most common stack 
and patterns, to begin with because I don't work alone. Furthermore, note some of my open-source contributions to major, common use projects with a more traditional stack.
* [Unbound concurrency exception fix on AndroidX Media3](https://github.com/androidx/media/pull/126).
* [New API for composite media sources on AndroidX Media3](https://github.com/androidx/media/pull/123).
* [Removal of unnecessary reflection on Moshi](https://github.com/square/moshi/pull/629/files)
* [Fix a bug on assisted-inject, now combined into Dagger2, with Kotlin data classes](https://github.com/cashapp/InflationInject/pull/148/)
