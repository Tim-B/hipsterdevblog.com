---
layout: post
title: "Building ZeroC Ice 3.5 projects with Gradle and IntelliJ"
date: 2015-04-21 19:35:35 +1000
comments: true
categories: 
 - Distributed Computing
 - Gradle
 - ZeroC Ice
 - IntelliJ
 - Java
---

[ZeroC Ice](https://zeroc.com/) is a distributed computing platform supporting many languages including Java. Building
an Ice project requires compiling "slice" data structure definitions into a compatible Java interface. Most often
it is recommended to use the [Eclipse plugin](https://zeroc.com/eclipse.html), however I prefer to use IntelliJ and 
a build tool which is IDE agnostic. Official Gradle support [is coming](https://doc.zeroc.com/display/Ice36/Gradle+Slice+Plug-in)
 in Ice 3.6, but that's still in beta. Fortunately it's quite easy to invoke te slice2java tool from Gradle and
 develop Ice 3.5 projects on Gradle and IntelliJ by extension.
 
<!-- more -->

# Project structure

You should set up your project using the standard Gradle structure:

```
src/main/java
src/main/slice
build.gradle
```

Put your Java sources in the `java` folder and your slice files in the `slice` folder.

# build.gradle

Populate your `build.gradle` using this base: 

{% codeblock lang:groovy build.gradle %}
apply plugin: 'java'

sourceSets {
    main {
        java {
            srcDir 'slice'
        }
    }
}

repositories {
    mavenLocal()
    mavenCentral()
    maven {
        url 'https://repo.zeroc.com/nexus/content/repositories/releases'
    }
}

dependencies {
    compile 'com.zeroc:ice:3.5.0'
    compile 'com.zeroc:icestorm:3.5.0'
}

task compileSlice(type: Exec) {
    def sliceDir = new File("${projectDir}/slice/main/java")
    sliceDir.mkdirs()

    commandLine 'slice2java', '--output-dir', "${projectDir}/slice/main/java", "${projectDir}/src/main/slice/MySliceFile.ice"

    standardOutput = new ByteArrayOutputStream()

    ext.output = {
        println standardOutput.toString()
    }

}

gradle.projectsEvaluated {
    compileJava.dependsOn(compileSlice)
}

String getRuntimeClasspath() {
    sourceSets.main.runtimeClasspath.collect { it.absolutePath }.join(':')
}
{% endcodeblock %}

Replace `MySliceFile.ice` with the name of your slice file.

# Building the project

You should now be able to run `gradle compileSlice` which will generate a new folder called `slice/main/java`,
compile the slice file and output the result to that new folder.

`compileSlice` is also a dependency of `compileJava` so you only have to run `gradle build` which will compile the
slice files then your Java source including the generated Java from the slice compiler in one go.

It is assumed that `slice2java` is in your path, if the build fails with a message such as 
`A problem occurred starting process 'command 'slice2java''` check that it's in your path by running `which slice2java`.

## Optional: Building distributable packages

You can alsop use the Gradle [application plugin](http://gradle.org/docs/current/userguide/application_plugin.html)
to generate distributable packages of your Ice application, including a bundled copy of `ice-3.5.0.jar`.
 
Update your `build.gradle`:

{% codeblock lang:groovy build.gradle %}
apply plugin: 'application'

// Include the base build.gradle content here

mainClassName = 'com.mydomain.myproject.MyClass'

// If you have multiple other components / entry classes 
// (eg. a client and a server that need to be invoked independently)

task createAllStartScripts() << {}

def scripts = ['MyOtherClass1' : 'com.mydomain.myproject.MyOtherClass1',
               'MyOtherClass2' : 'com.mydomain.myproject.MyOtherClass2']

scripts.each() { scriptName, className ->
    def t = tasks.create(name: scriptName + 'StartScript', type: CreateStartScripts) {
        mainClassName = className
        applicationName = scriptName
        outputDir = new File(project.buildDir, 'scripts')
        classpath = jar.outputs.files + project.configurations.runtime
    }
    applicationDistribution.into("bin") {
        from(t)
        fileMode = 0755
    }
    createAllStartScripts.dependsOn(t)
}

// If you want to include other files with your distributable

applicationDistribution.from(new File("${projectDir}/config")) {
    into "bin/config"
}

applicationDistribution.from(new File("${projectDir}/data")) {
    into "bin/data"
}

{% endcodeblock %}

Substitute your own class names as applicable in the code above.

Build your distributable by running `gradle distZip`, you should find the output under `build/distributions/MyProject.zip`.

# Using IntelliJ

You should now be able to import your Gradle project into IntelliJ like any other Gradle project: `File` > `New` >
`Project from Existing Sources...` > browse to and select `build.gradle`. 

Create a new gradle run configuration for the `gradle build` task:

{% img /images/posts/icegradle/build.png %}

Also add a run configuration to run your application:

{% img /images/posts/icegradle/run.png %}

You should now be able to both build and run your application directly from IntelliJ.

## Caveat on OSX

It's a [well](http://apple.stackexchange.com/questions/51677/how-to-set-path-for-finder-launched-applications) 
[documented](http://emmanuelbernard.com/blog/2012/05/09/setting-global-variables-intellij/)
 [issue](http://stackoverflow.com/questions/15201763/intellij-does-not-recognize-path-) that IntelliJ does not inherit 
 the path variable of the user on OSX. This means that when
you run `gradle build` from IntelliJ it won't be able to find `slice2java` and you'll get an error like
`A problem occurred starting process 'command 'slice2java''`. The easiest solution is just to use an absolute path 
in `commandLine 'slice2java'` such as `commandLine '/usr/bin/slice2java'`, alternatively you can explore some of the 
solutions mentioned in the links above.





