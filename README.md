# Pipeline Shared Libraries

When you have multiple Pipeline jobs, you often want to share some parts of the Pipeline
scripts between them to keep Pipeline scripts [DRY](http://en.wikipedia.org/wiki/Don't_repeat_yourself).
A very common use case is that you have many projects that are built in the similar way.

This plugin adds that functionality by allowing you to create “shared library script” SCM repositories.
It can be used in two modes:
a legacy mode in which there is a single Git repository hosted by Jenkins itself, to which you may push changes;
and a more general mode in which you may define libraries hosted by any SCM in a location of your choice.

## Directory structure

The directory structure of a shared library repository is as follows:

    (root)
     +- src                     # Groovy source files
     |   +- org
     |       +- foo
     |           +- Bar.groovy  # for org.foo.Bar class
     +- vars
     |   +- foo.groovy          # for global 'foo' variable/function
     |   +- foo.txt             # help for 'foo' variable/function
     +- resources               # resource files (external libraries only)
     |   +- org
     |       +- foo
     |           +- bar.json    # static helper data for org.foo.Bar

The `src` directory should look like standard Java source directory structure.
This directory is added to the classpath when executing Pipelines.

The `vars` directory hosts scripts that define global variables accessible from
Pipeline scripts.
The basename of each `*.groovy` file should be a Groovy (~ Java) identifier, conventionally `camelCased`.
The matching `*.txt`, if present, can contain documentation, processed through the system’s configured markup formatter
(so may really be HTML, Markdown, etc., though the `txt` extension is required).

The Groovy source files in these directories get the same “CPS transformation” as your Pipeline scripts.

A `resources` directory allows the `libraryResource` step to be used from an external library to load associated non-Groovy files.
Currently this feature is not supported for internal libraries.

Other directories under the root are reserved for future enhancements.

## Defining external libraries

An external library is defined with a name, a source code retrieval method such as by SCM, and optionally a default version.
The name should be a short identifier as it will be used in scripts.

The version could be anything understood by that SCM; for example, branches, tags, and commit hashes all work for Git.
You may also declare whether scripts need to explicitly request that library (detailed below), or if it is present by default.
Furthermore, if you specify a version in Jenkins configuration, you can block scripts from selecting a _different_ version.

The best way to specify the SCM is using an SCM plugin which has been specifically updated
to support a new API for checking out an arbitrary named version (_Modern SCM_ option).
As of this writing, the latest versions of the Git and Subversion plugins support this mode; others should follow.

If your SCM plugin has not been integrated, you may select _Legacy SCM_ and pick anything offered.
In this case, you need to include `${library.yourLibName.version}` somewhere in the configuration of the SCM,
so that during checkout the plugin will expand this variable to select the desired version.
For example, for Subversion, you can set the _Repository URL_ to `https://svnserver/project/${library.yourLibName.version}`
and then use versions such as `trunk` or `branches/dev` or `tags/1.0`.

### Global external libraries

There are several places you can define external libraries, according to your needs.

Under _Manage Jenkins » Configure System » Global Pipeline Libraries_ you may configure as many libraries as you need.
These will be accessible to any Pipeline job in the system.

These libraries are trusted: they can run any methods in Java, Groovy, Jenkins internal APIs, Jenkins plugins, or third-party libraries.
This allows you to define libraries which encapsulate individually unsafe APIs in a higher-level wrapper safe for use from any job.
Beware that **anyone able to push commits to this SCM repository could obtain unlimited access to Jenkins**.
You need the _Overall/RunScripts_ permission to configure these libraries (normally this will be granted to Jenkins administrators).

### Folder-level external libraries

When you create a folder, you can associate some libraries with it.
These will be available to any job running inside the folder (or a subfolder).

Folder-based libraries are not trusted: they run in the Groovy sandbox just like typical Pipeline scripts.

### Automatic external libraries

Other plugins may add ways of defining libraries on the fly just based on name.
For example, the _GitHub Organization Folder_ plugin allows a script to use an (untrusted) library
named something like `github.com/someorg/somerepo` without any Jenkins configuration at all:
the specified GitHub repository is loaded, by default in the `master` branch, using an anonymous checkout.

## Defining the internal library

An older mode of sharing code is to deploy sources to Jenkins itself.
This special library is trusted (no Groovy sandbox),
since the _Overall/RunScripts_ permission is required to push any changes to the global library repository,

This library is implicitly available to all scripts.
There is no way for a script to select a particular version to use; the `master` branch is always loaded.

This directory is managed by Git, and you'll deploy new changes through `git push`.
The repository is exposed in two endpoints:

 * `ssh://USERNAME@server:PORT/workflowLibs.git` through [Jenkins SSH](https://wiki.jenkins-ci.org/display/JENKINS/Jenkins+SSH)
 * `http://LOCATION/workflowLibs.git` (when your Jenkins app is located on the url `http://LOCATION/`). As noted in [JENKINS-26537](https://issues.jenkins-ci.org/browse/JENKINS-26537), this mode will not currently work in an authenticated Jenkins instance. 

Having the shared library script in Git allows you to track changes, perform
tested deployments, and reuse the same scripts across a large number of instances.

Note that the repository is initially empty of any commits, so it is possible to push an existing repository here.
Normally you would instead `clone` it to get started, in which case Git will note

    warning: remote HEAD refers to nonexistent ref, unable to checkout.

To set things up after cloning, start with:

    git checkout -b master

Now you may add and commit files normally.
For your first push to Jenkins you will need to set up a tracking branch:

    git push --set-upstream origin master

Thereafter it should suffice to run:

    git push

## Using libraries

Pipeline scripts need do nothing special to access external libraries marked _Load implicitly_,
or the legacy internal library.
They may immediately use classes or global variables defined by any such libraries (details below).

To access other external libraries, a script needs to use the `@Library` annotation.
It can take a library name:

```groovy
@Library('somelib')
```

or a library with a version specifier (branch, tag, etc.):

```groovy
@Library('somelib@1.0')
```

or several libraries:

```groovy
@Library(['somelib', 'otherlib@abc1234'])
```

The annotation can be anywhere in the script where an annotation is permitted by Java/Groovy.
When referring to class libraries (with `src/` directories), conventionally the annotation goes on an `import` statement:

```groovy
@Library('somelib')
import com.mycorp.pipeline.somelib.UsefulClass
```

It is legal, though unnecessary, to `import` a global variable (or function) defined in a `vars/` directory:

```groovy
@Library('somelib')
import usefulFunction
```

If you have nowhere better to put it, the simplest legal syntax is an unused, untyped field named `_`:

```groovy
@Library('somelib') _
```

Note that libraries are resolved and loaded during compilation of the script, before it starts running.
This allows the Groovy compiler to understand the meaning of symbols you use in static type checking,
and permits them to be used in type declarations in your script:

```groovy
@Library('somelib')
import com.mycorp.pipeline.somelib.Helper

int useSomeLib(Helper helper) {
    helper.prepare()
    return helper.count()
}

echo useSomeLib(new Helper('some text'))
```

This matters less for global variables/functions, which are resolved at runtime.

### Overriding versions

A `@Library` annotation may override a default version given in the library’s definition, if the definition permits this.
In particular, an external library marked for implicit use can still be loaded in a different version using the annotation
(unless the definition specifically forbids this).

## Writing libraries

Whether external or internal, shared libraries offer several mechanisms for code reuse.

### Writing shared code
At the base level, any valid Groovy code is OK. So you can define data structures,
utility functions, and etc., like this:

```groovy
// src/org/foo/Point.groovy
package org.foo;

// point in 3D space
class Point {
  float x,y,z;
}
```

#### Accessing steps

Library classes cannot directly call step functions like `sh` or `git`.
You might want to define a series of functions that in turn invoke other Pipeline step functions.
You can do this by not explicitly defining the enclosing class,
just like your main Pipeline script itself:

```groovy
// src/org/foo/Zot.groovy
package org.foo;

def checkOutFrom(repo) {
  git url: "git@github.com:jenkinsci/${repo}"
}
```

You can then call such function from your main Pipeline script like this:

```groovy
def z = new org.foo.Zot()
z.checkOutFrom(repo)
```

However this style has its own limitations; for example, you cannot declare a superclass.

Alternately, you can explicitly pass the set of `steps` to a library class, in a constructor or just one method:

```groovy
package org.foo

// This class must extend Serializable because it has a non-static data member, "steps".
class Utilities implements Serializable {
  def steps
  Utilities(steps) {this.steps = steps}
  def mvn(args) {
    steps.sh "${steps.tool 'Maven'}/bin/mvn -o ${args}"
  }
}
```

which might be accessed like this from a script:

```groovy
@Library('utils') import org.foo.Utilities
def utils = new Utilities(steps)
node {
  utils.mvn 'clean package'
}
```

If you need to access `env` or other global variables as well,
you could pass these in explicitly in the same way,
or simply pass the entire top-level script rather than just `steps`:

```groovy
package org.foo

// This class only has static members so, unlike the previous example
// it doesn't need to extend Serializable.
class Utilities { 
  static def mvn(script, args) {
    script.sh "${script.tool 'Maven'}/bin/mvn -s ${script.env.HOME}/jenkins.xml -o ${args}"
  }
}
```

The above example shows the script being passed in to one `static` method, so it could be accessed like this:

```groovy
@Library('utils') import static org.foo.Utilities.*
node {
  mvn this, 'clean package'
}
```

### Defining global functions
You can define your own functions that looks and feels like built-in step functions like `sh` or `git`.
For example, to define `helloWorld` step of your own, create a file named `vars/helloWorld.groovy` and
define the `call` method:

```groovy
// vars/helloWorld.groovy
def call(name) {
    // you can call any valid step functions from your code, just like you can from Pipeline scripts
    echo "Hello world, ${name}"
}
```

Then your Pipeline can call this function like this:

```groovy
helloWorld "Joe"
helloWorld("Joe")
```

If called with a block, the `call` method will receive a `Closure` object. You can define that explicitly
as the type to clarify your intent, like the following:

```groovy
// vars/windows.groovy
def call(Closure body) {
    node('windows') {
        body()
    }
}
```

Your Pipeline can call this function like this:

```groovy
windows {
    bat "cmd /?"
}
```

See [the closure chapter of Groovy language reference](http://www.groovy-lang.org/closures.html) for more details
about the block syntax in Groovy.

### Defining global variables
Internally, scripts in the `vars` directory are instantiated as a singleton on-demand, when used first.
So it is possible to define more methods, properties on a single file that interact with each other:

```groovy
// vars/acme.groovy
def setFoo(v) {
    this.foo = v;
}
def getFoo() {
    return this.foo;
}
def say(name) {
    echo "Hello world, ${name}"
}
```

Then your Pipeline can call these functions like this:

```groovy
acme.foo = "5";
echo acme.foo; // print 5
acme.say "Joe" // print "Hello world, Joe"
```

Note that a variable defined in an external library will currently only show up in _Global Variables Reference_ (under _Pipeline Syntax_)
after you have first run a successful build using that library, allowing its sources to be checked out by Jenkins.

### Define more structured DSL
If you have a lot of Pipeline jobs that are mostly similar, the global function/variable mechanism gives you
a handy tool to build a higher-level DSL that captures the similarity. For example, all Jenkins plugins are
built and tested in the same way, so we might write a global function named `jenkinsPlugin` like this:

```groovy
// vars/jenkinsPlugin.groovy
def call(body) {
    // evaluate the body block, and collect configuration into the object
    def config = [:]
    body.resolveStrategy = Closure.DELEGATE_FIRST
    body.delegate = config
    body()

    // now build, based on the configuration provided
    node {
        git url: "https://github.com/jenkinsci/${config.name}-plugin.git"
        sh "mvn install"
        mail to: "...", subject: "${config.name} plugin build", body: "..."
    }
}
```

With this as an internal or implicit external library,
the Pipeline script will look a whole lot simpler,
to the point that people who know nothing about Groovy can write it:

```groovy
jenkinsPlugin {
    name = 'git'
}
```

### Using third-party libraries

You may use third-party Java libraries (typically found in Maven Central) within *trusted* library code via `@Grab`.
Refer to the [Grape documentation](http://docs.groovy-lang.org/latest/html/documentation/grape.html#_quick_start) for details.
For example:

```groovy
@Grab('org.apache.commons:commons-math3:3.4.1')
import org.apache.commons.math3.primes.Primes
void parallelize(int count) {
  if (!Primes.isPrime(count)) {
    error "${count} was not prime"
  }
  // …
}
```

Libraries are cached by default in `~/.groovy/grapes/` on the Jenkins master.

### Loading resources

External libraries may load adjunct files from a `resources/` directory using the `libraryResource` step.
The argument is a relative pathname, akin to Java resource loading:

```groovy
def request = libraryResource 'com/mycorp/pipeline/somelib/request.json'
```

The file is loaded as a string, suitable for passing to certain APIs or saving to a workspace using `writeFile`.

It is advisable to use an unique package structure so you do not accidentally conflict with another library.

### Pretesting library changes

If you notice a mistake in a build using an untrusted library,
simply click the _Replay_ link to try editing one or more of its source files,
and see if the resulting build behaves as expected.
Once you are satisfied with the result, follow the diff link from the build’s status page,
and apply the diff to the library repository and commit.

(Even if the version requested for the library was a branch, rather than a fixed version like a tag,
replayed builds will use the exact same revision as the original build:
library sources will not be checked out again.)

_Replay_ is not currently supported for trusted libraries.
Modifying resource files is also not currently supported during _Replay_.
