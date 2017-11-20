# kscript - Having fun with Kotlin scripting

[![Build Status](https://travis-ci.org/holgerbrandl/kscript.svg?branch=master)](https://travis-ci.org/holgerbrandl/kscript)

Enhanced scripting support for [Kotlin](https://kotlinlang.org/) on *nix-based systems.

Kotlin has some built-in support for scripting already but it is not yet feature-rich enough to be a viable alternative in the shell.

In particular this wrapper around `kotlinc` adds
* Compiled script caching (using md5 checksums)
* Dependency declarations using gradle-style resource locators and automatic dependency resolution with maven
* More options to provide scripts including interpreter mode, reading from stdin, local files or URLs
* Embedded configuration for Kotlin runtime options
* Support library to ease the writing of Kotlin scriptlets

Taken all these features together, `kscript` provides an easy-to-use, very flexible, and almost zero-overhead solution to write self-contained mini-applications with Kotlin.


---
### [`kscript` presentation from KotlinConf2017!](https://holgerbrandl.github.io/kscript_kotlinconf_2017/kscript_kotlinconf.html)
---

Installation
------------

To use `kscript` just Kotlin and Maven are required. To [install Kotlin](https://kotlinlang.org/docs/tutorials/command-line.html) we recommend [sdkman](http://sdkman.io/install.html):
```
curl -s "https://get.sdkman.io" | bash  # install sdkman
source ~/.bash_profile                  # add sdkman to PATH

sdk install kotlin                      # install Kotlin
sdk install maven                       # install Maven
```

Once Maven and Kotlin are ready, you can install `kscript` with
```
sdk install kscript
```


To test your installation simply run
```bash
kscript --help
```

`kscript` can update itself to its latest version by running
```bash
kscript --self-update
```

#### Installation without `sdkman`

If you have Kotlin and Maven already and you would like to install the latest `kscript` release without using `sdkman` you can do so by unzipping the [latest ](https://github.com/holgerbrandl/kscript/releases/latest) binary release. Don't forget to update your `$PATH` accordingly.

#### Installation with Homebrew

On MacOS you can install `kscript` also with [Homebrew](https://brew.sh/)
```
brew install holgerbrandl/tap/kscript
```

#### Build it yourself

To build `kscript` yourself, simply clone the repo and do
```bash
gradle shadowJar
cp build/libs/kscript-0.1-SNAPSHOT-all.jar kscript.jar

## Don't forget to prefix with `./` when running the freshly compiled version 
./kscript
```

## Script Input Modes

The main mode of operation is `kscript <script>`.

The `<script>` can be a Kotlin `*.kts` script file , a script URL, `-` for stdin, a process substitution file handle, a `*.kt` source file with a main method, or some kotlin code.


### Interpreter Usage

To use `kscript` as interpreter for a script just point to it in the shebang line of your Kotlin scripts:

```kotlin
#!/usr/bin/env kscript

println("Hello from Kotlin!")
for (arg in args) {
    println("arg: $arg")
}
```

### Inlined Usage


To use `kscript` in a workflow without creating an additional script file, you can also use one of its supported modes for _inlined usage_. The following modes are supported:

* Directly provide a Kotlin scriptlet as argument
```{bash}
kscript 'println("hello world")'
```


* Pipe a Kotlin snippet into `kscript` and instruct it to read from `stdin` by using `-` as script argument

```{bash}
echo '
println("Hello Kotlin.")
' |  kscript -
```


* Using `heredoc` (preferred solution for inlining) which gives you some more flexibility to also use single quotes in your script:
```{bash}
kscript - <<"EOF"
println("It's a beautiful day!")
EOF
```

* Since the piped content is considered as a regular script it can also have dependencies
```{bash}
kscript - <<"EOF"
//DEPS com.offbytwo:docopt:0.6.0.20150202 log4j:log4j:1.2.14

import org.docopt.Docopt
val docopt = Docopt("Usage: jl <command> [options] [<joblist_file>]")

println("hello again")
EOF
```

* Finally (for sake of completeness), it also works with process substitution and for sure you can always provide additional arguments (exposed as `args : Array<String>` within the script)
```{bash}
kscript <(echo 'println("k-onliner")') arg1 arg2 arg3 
```

Inlined _kscripts_ are also cached based on `md5` checksum, so running the same snippet again will use a cached jar (sitting in `~/.kscript`).


### URL usage

To support remote scriplet repositories, `kscript` can also work with URLs. Consider the following [hello-world-gist-scriptlet](https://github.com/holgerbrandl/kscript/blob/master/examples/url_example.kts) which is hosted on github (but any URL would work). To run it locally as a tool simply refer to it (here using the shortened [raw-URL](https://raw.githubusercontent.com/holgerbrandl/kscript/master/examples/url_example.kts) of the script for better readability)

```bash
kscript https://git.io/v1cG6 my argu ments 
```

To streamline the usage, the first part could be even aliased:
```bash
alias hello_kscript="kscript https://git.io/v1cG6"
hello_kscript my argu ments
```
Via this mechanism, `kscript` allows for easy integration of remotely hosted (mini) programs into data workflows.

URL-scripts are cached locally to speed up processing, and `kscript --clear-cache` can be used to wipe the cache if needed.

See this [blogpost](http://holgerbrandl.github.io/kotlin/2016/12/02/mini_programs_with_kotlin.html) for a more extensive overview about URL support in `kscript`.

Script Configuration
--------------------

The following directives supported by `kscript` to configure scripts:

* `//DEPS` to declare dependencies with gradle-style locators
* `//KOTLIN_OPTS`  to configure the kotlin/java runtime environment
* `//INCLUDE` to source kotlin files into the script
* `//ENTRY` to declare the application entrypoint for kotlin `*.kt` applications


### Declare dependencies with `//DEPS`

To specify dependencies simply use gradle-style locators. Here's an example using [docopt](https://github.com/docopt/docopt.java) and [log4j](http://logging.apache.org/log4j/2.x/)

```kotlin
#!/usr/bin/env kscript
//DEPS com.offbytwo:docopt:0.6.0.20150202,log4j:log4j:1.2.14

import org.docopt.Docopt
import java.util.*


val usage = """
Use this cool tool to do cool stuff
Usage: cooltool.kts [options] <igenome> <fastq_files>...

Options:
 --gtf <gtfFile>     Custom gtf file instead of igenome bundled copy
 --pc-only           Use protein coding genes only for mapping and quantification
"""

val doArgs = Docopt(usage).parse(args.toList())

println("Hello from Kotlin!")
println("Parsed script arguments are: \n" + doArgs)
```

`kscript` will read dependencies from all lines in a script that start with `//DEPS` (if any). Multiple dependencies can be split by comma, space or semicolon.

### Configure the runtime  with `//KOTLIN_OPTS`

`kscript` allows to provide a `//KOTLIN_OPTS` directive followed by parameters passed on to `kotlin` similar to how dependencies are defined:
```kotlin
#!/usr/bin/env kscript
//KOTLIN_OPTS -J-Xmx5g  -J-server

println("Hello from Kotlin with 5g of heap memory running in server mode!")
```

### Ease prototyping with `//INCLUDE`

`kscript` supports an `//INLCUDE` directive to directly include other source files without prior compilation. Absolute and relative paths, as well as URLs are supported. Example:

```kotlin
//utils.kt
fun Array<Double>.median(): Double {
    val (lower, upper) = sorted().let { take(size / 2) to takeLast(size / 2) }
    return if (size % 2 == 0) (lower.last() + upper.first()) / 2.0 else upper.first()
}
```

Which can be now used using the `//INCLUDE` directive with

```kotlin
#!/usr/bin/env kscript

//INCLUDE utils.kt

val robustMean = listOf(1.3, 42.3, 7.).median()
println(robustMean)
```
The argument file can be an url, absolute or relative path.

For more examples see [here](test/resources/includes/include_variations.kts).


### Use `//ENTRY` to run applications with `main` method


`kscript` also supports running regular Kotlin `kt` files.

Example: `./examples/Foo.kt`:

```kotlin
package examples

//ENTRY examples.Bar

class Bar{
    companion object {
        @JvmStatic fun main(args: Array<String>) {
            println("Foo was called")
        }
    }
}

fun main(args: Array<String>) =  println("main was called")
```

To run top-level main instead we would use `//ENTRY examples.FooKt`

The latter is the default for `kt` files and could be omitted


### Annotation driven script configuration

Using annotations instead of comment directives to configure scripts is cleaner and allow for better tooling.

```kotlin
// annotation-driven script configuration
@file:DependsOn("de.mpicbg.scicomp:kutils:0.4")

// comment directive
//DEPS de.mpicbg.scicomp:kutils:0.4
```

 To do so `kscript` supports [annotations](https://github.com/holgerbrandl/kscript_annotations) to be used instead of comment directives:
```kotlin
#!/usr/bin/env kscript

// Declare dependencies
@file:DependsOn("de.mpicbg.scicomp:kutils:0.4")
@file:DependsOn("com.beust:klaxon:0.24", "com.github.kittinunf.fuel:fuel:1.3.1")


// To use a custom maven repository you can declare it with
@file:MavenRepository("imagej-releases","http://maven.imagej.net/content/repositories/releases" )

// For compatibility with https://github.com/ligee/kotlin-jupyter kscript supports also
@file:DependsOnMaven("net.clearvolume:cleargl:2.0.1")
// Note that for compatibility reasons, only one locator argument is allowed for @DependsOnMaven


// Include helper scripts without deployment or prior compilation
@file:Include("util.kt")


// Define kotlin options
@file:KotlinOpts("-J-Xmx5g")
@file:KotlinOpts("-J-server")

// declare application entry point (applies on for kt-files)
@file:EntryPoint("Foo.bar") 

print("1+1")
```

To enable the use of these annotations in Intellij, the user must add the following artifact (hosted on jcenter) to the project dependencies:
```
com.github.holgerbrandl:kscript-annotations:1.1
```

`kscript` will automatically detect an annotation-driven script, and if so will declare a dependency on this artifact internally.

Support API
-----------


`kscript` is complemented by a [support library](https://github.com/holgerbrandl/kscript-support-api) to ease the writing of Kotlin scriptlets for text-processing. The latter includes solutions to common use-cases like argument parsing, data streaming, IO utilities, and various iterators to streamline the development of kscript applications.

The text processing mode can be enabled with `-t` or `--text`. If so, `kscript` will
* declare `com.github.holgerbrandl:kscript:1.2.3` as dependency for the script
* import the  `kscript.*` namespace
* Define variable `val lines = kscript.text.StreamUtilKt.resolveArgFile(args)` which returns an iterator over the lines in the first input argument of the script, or the standard input if no file arguments are provided to the script

This allows for `sed`/`awk`/`perl`-like constructs such as

```bash
cat some_file | kscript -t 'lines.filter { "^de0[-0]*".toRegex().matches(it) }.map { it + "foo:" }.print()'
```

In this example, the extension method [`Iterable<String>.print()`](https://github.com/holgerbrandl/kscript-support-api/blob/master/src/main/kotlin/kscript/text/StreamUtil.kt#L34) to print the lines to stdout comes from the support API. The rest is stdlib Kotlin.

 For more  examples using the support library see this [blog post](http://holgerbrandl.github.io/kotlin/2017/05/08/kscript_as_awk_substitute.html).



FAQ
---

### Why is `kscript` not calling the main method in my `.kts` script?

There is [no need](https://kotlinlang.org/docs/tutorials/command-line.html#using-the-command-line-to-run-scripts) for a `main` method in a Kotlin script. Kotlin `*.kts` scripts can be more simplistic compared to more common kotlin `*.kt` source files. The former work without a `main` method by directly running the provided code from top to bottom. E.g.
```
print("hello kotlin!")
```
is a valid Kotlin `kts` script. Plain and simple, no `main`, no `companion`, just a few bits of code.


### Ok, but does `kscript` also work for regular kotlin `.kt` source files  with a `main` as entry point?

Yes, (since v1.6) you can run kotlin source files through `kscript`. By default it will assume a top-level `main` method as entry-point.

However in case you're using a companion object to declare the entry point, you need to indicate this via the `//ENTRY` directive:


### Why does it fail to read my script file when using cygwin?

In order to use cygwin you need to use windows paths to provide your scripts. You can map cygwin paths using `cygpath`. Example
```bash
kscript $(cygpath -w /cygdrive/z/some/path/my_script.kts)
```


### Can I use custom artifact repositories?

Yes, via the `@MavenRepository` annotation. See [annotations section](#annotation-driven-script-configuration) or [custom_mvn_repo_annot](test/resources/custom_mvn_repo_annot.kts) for a complete example


Support
-------

Feel welcome to post ideas and suggestions to our [tracker](https://github.com/holgerbrandl/kscript/issues).


How to contribute?
------------------

We always welcome pull requests. :-)

You could also show your support by upvoting `kscript` here on github, or by voting for issues in Intellij IDEA which impact  `kscript`ing. Here are our top 3 tickets/annoyances that we would love to see fixed:

1. [KT-13347](https://youtrack.jetbrains.com/issue/KT-13347) Good code is red in injected kotlin language snippets
2. [KT-16802](https://youtrack.jetbrains.com/issue/KT-16802) Good code is always red when editing kts-script files under Kotlin v1.1
3. [KT-10368](https://youtrack.jetbrains.com/issue/KT-10368) Kotlin scratch files not runnable?



Acknowledgements
----------------

The initial version of `kscript` was kindly contributed by [Oscar Gonzalez](https://github.com/oskargb).

Thanks also to the [Scionics Computer Innovation GmbH](https://www.scionics.com/) and the [MPI-CBG](http://www.mpi-cbg.de) for supporting this project.

`kscript` was inspired by [kotlin-script](https://github.com/andrewoma/kotlin-script) which is another great way to do scripting in Kotlin.

