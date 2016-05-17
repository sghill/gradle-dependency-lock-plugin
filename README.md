gradle-dependency-lock-plugin
=============================

![Support Status](https://img.shields.io/badge/nebula-supported-brightgreen.svg)
[![Build Status](https://travis-ci.org/nebula-plugins/gradle-dependency-lock-plugin.svg?branch=master)](https://travis-ci.org/nebula-plugins/gradle-dependency-lock-plugin)
[![Coverage Status](https://coveralls.io/repos/nebula-plugins/gradle-dependency-lock-plugin/badge.svg?branch=master&service=github)](https://coveralls.io/github/nebula-plugins/gradle-dependency-lock-plugin?branch=master)
[![Gitter](https://badges.gitter.im/Join%20Chat.svg)](https://gitter.im/nebula-plugins/gradle-dependency-lock-plugin?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge)
[![Apache 2.0](https://img.shields.io/github/license/nebula-plugins/gradle-dependency-lock-plugin.svg)](http://www.apache.org/licenses/LICENSE-2.0)

A plugin to allow people using dynamic dependency versions to lock them to specific versions.

Some project teams may prefer to have their build.gradle dependencies reflect their ideal world. A latest.release for internal dependencies. A major.+, major.minor.+, or a range \[2.0.0, 4.0.0\). Many also want to lock to specific versions for day to day development, having a tagged version always resolve identically, and for published versions to have specific dependencies.

Inspired by [Bundler](http://bundler.io)

### General Use Pattern

1. Daily locking of dynamic dependencies
    a. Run a daily job to lock your dependencies. Every day before you show up to work you can have an automated system to pull in the latest dependencies and run your tests. If they pass, your team has a stable dependency graph for the day. If they fail you can have a subset of the team deal with the dependency issues while the rest keep with what they're were doing using yesterday's dependency lock.
    b. Occasionally (and optionally) run `updateLock` to try out a single new dependency.
2. Lock on release
    a. When you're ready to cut that shiny new version of your software create a lock of your dependencies. You'll be able to recreate the exact state of your dependency graph if you ever need to reproduce a build.
3. Deal with IDE issues
    a. Create a global lock with `generateGlobalLock` `saveGlobalLock`.
    b. Watch as you no longer have to deal with one subproject depend on an old version and mess up another's dependency graph
    c. Stop WAR deployments from copying 2 different versions of a dependency into the directory.

## Deprecation Warning

The old plugin name `gradle-dependency-lock`/`nebula.gradle-dependency-lock` has been deleted in favor of `dependency-lock`/`nebula.dependency-lock`.

## Usage

### Applying the Plugin

To include, add the following to your build.gradle

If newer than gradle 2.1 you may use

    plugins {
      id 'nebula.dependency-lock' version '4.2.0'
    }

*or*

    buildscript {
      repositories { jcenter() }

      dependencies {
        classpath 'com.netflix.nebula:gradle-dependency-lock-plugin:4.2.0'
      }
    }

    apply plugin: 'nebula.dependency-lock'

### Tasks Provided

Command line overrides via `-PdependencyLock.override` or `-PdependencyLock.overrideFile` will apply.

| Task | Description |
| --- | --- |
| `generateLock` | Generate a lock file into the build directory. Any existing `dependencies.lock` file will be ignored. |
| `updateLock` | Update dependencies from the lock file into the build directory. By default, this task does the same thing as the `generateLock` task. If updateDependencies is specified in the extension or `-PdependencyLock.updateDependencies` on the command line we will keep direct dependencies not in the list locked and allow conflict resolution on everything else. |
| `saveLock` | Copy the generated lock into the project directory. |
| `deleteLock` | Delete the existing lock files. |
| `generateGlobalLock` | Generate a lock file into the build directory representing the global dependencies of the entire multiproject. Any existing `dependencies.lock` or `global.lock` will be ignored. |
| `updateGlobalLock` | Update dependencies from the lock file into the build directory. By default, this task does the same thing as the `generateGlobalLock` task. |
| `saveGlobalLock` | Copies the generated globalLock into the project directory. |
| `deleteGlobalLock` | Delete the `global.lock` file. |
| `commitLock` | If a [gradle-scm-plugin](https://github.com/nebula-plugins/gradle-scm-plugin) implementation is applied. Will commit dependencies.lock to the configured SCM. Exists only on the rootProject. Assumes scm root is at the same level as the root build.gradle. |

### Notes About Global vs Project Locks

* If a `global.lock` is found it will be used, ignoring all `dependencies.lock` files.
* `saveLock` will fail if you have a `global.lock` -- You should run `deleteGlobalLock`
* `saveGlobalLock` will fail if you have any `dependencies.lock` files -- You should run `deleteLock`

#### Common Command Line Overrides

Revert to normal gradle behavior even with plugin applied.

    ./gradlew -PdependencyLock.ignore=true <tasks>

### Common Workflows

Generate lock:

1. `./gradlew generateLock saveLock`
2. `./gradlew test`
3. if 2 passes `./gradlew commitLock`

or

1. `./gradlew generateLock`
2. `./gradlew -PdependencyLock.useGeneratedLock=true test`
3. `./gradlew saveLock commitLock`

Update lock (the lock must still be saved/committed):

* `./gradlew updateLock -PdependencyLock.updateDependencies=com.example:foo,com.example:bar`

### Extensions Provided

#### dependencyLock Extension

| Property | Description |
| --- | --- |
| `lockFile` | This field takes a String. The default is `dependencies.lock`. This filename will be what is generated by `generateLock` and read when locking dependencies. |
| `configurationNames` | This field takes a List<String>. These will be the configurations that are written to the lock file when locking. Defaults to all configurations. |
| `dependencyFilter` | This field can be assigned a Closure that is used to filter the set of top-level dependencies as they are retrieved from the configurations. This happens before overrides are applied and before any dependencies are skipped. The Closure must accept the dependency's `group`, `name`, and `version` as its 3 parameters. The default implementation returns `true`, meaning all dependencies are used. |
| `updateDependencies` | This field takes a List<String> denoting the dependencies that should be updated when the `updateLock` task is run. If any dependencies are specified via the `--dependencies` option, this field is ignored. If any dependencies are listed during execution of the `updateLock` task either via the `--dependencies` option or this field, the `dependencyFilter` is bypassed. |
| `skippedDependencies` | This field takes a List<String>. Defaults to empty. This list is used to list dependencies as ones that will never be locked. Strings should be of the format `<group>:<artifact>` |
| `includeTransitives` | This field is a boolean. Defaults to false. False will only lock direct dependencies. True will lock the entire transitive graph. |

Use the extension if you wish to configure. Each project where gradle-dependency-lock is applied will have its own dependencyLock extension.
The following values are the defaults. If they work for you, you can skip configuring the plugin.

    dependencyLock {
      lockFile = 'dependencies.lock'
      globalLockFile = 'global.lock'
      dependencyFilter = { String group, String name, String version -> true }
      updateDependencies = []
      skippedDependencies = []
      includeTransitives = false
      lockAfterEvaluating = true
    }

#### commitDependencyLock Extension

*Properties*

* commitMessage - Commit message to use.
* shouldCreateTag - Boolean to tell the commitLock to create a tag, defaults to false.
* tag - A 0 argument closure that returns a String. Needs to generate a unique tag name.
* remoteRetries - Number of times to update from remote repository and retry commits.

Use the following to configure. There will be only one commitDependencyLock extension attached to the rootProject in a multiproject.  

    commitDependencyLock {
      message = 'Committing dependency lock files'
      shouldCreateTag = false
      tag = { "LockCommit-${new Date().format('yyyyMMddHHmmss')}" }
      remoteRetries = 3
    }

### Properties that Affect the Plugin

In general use of properties should be minimal, usually you want to create a custom task that wires your options.

*dependencyLock.lockFile*

Allows the user to override the configured lockFile name via the command line. It will expect to find these files in the project directories.

    ./gradlew -PdependencyLock.lockFile=<filename> <tasks>

*dependencyLock.globalLockFile*

Allows the user to override the configured globalLockFile name via the command line. It will expect to find this file in the root project directory (where the settings.gradle lives)

    ./gradlew -PdependencyLock.globalLockFile=<filename> <tasks>

*dependencyLock.ignore*

Allows the user to ignore any present lockFile and/or command line overrides falling back to standard gradle dependency
resolution. Plugin checks for whether this is set to something that will resolve to true.

    ./gradlew -PdependencyLock.ignore=true <tasks>

*dependencyLock.includeTransitives*

Allows the user to set if transitive dependencies should be included in the lock file.

    ./gradlew -PdependencyLock.includeTransitives=true <tasks>

*dependencyLock.useGeneratedLock*

Use generated lock files in the build directory instead of the locks in the project directories.

    ./gradlew -PdependencyLock.useGeneratedLock=true <task>

*dependencyLock.overrideFile*

Allows the user to specify a file of overrides. This file should be formatted as specified in the 
File Format section. These will override the locked values in the dependencies.lock file. They will be respected when
running generateLock. This file is expected at the top level project.

    ./gradlew -PdependencyLock.overrideFile=override.lock <tasks>

*dependencyLock.override*

Allows the user to specify overrides to libraries on the command line. This override will be used over any from `dependencyLock.overrideFile`

    ./gradlew -PdependencyLock.override=group:artifact:version <tasks>

or to override multiple libraries

    ./gradlew -PdependencyLock.override=group0:artifact0:version0,group1:artifact1:version1 <tasks>
    
*dependencyLock.updateDependencies*

Allows user to list the dependencies that will be used by updateLock or updateGlobalLock

    ./gradlew updateLock -PdependencyLock.updateDependencies=com.example:foo,com.example:bar

*commitDependencyLock.message*

Allows the user to override the commit message.

    ./gradlew -PcommitDependencyLock.message='commit message' <tasks> commitLock

*commitDependencyLock.tag*

Allows the user to specify a String for the tagname. If present commitLock will tag the commit with the given String.

    ./gradlew -PcommitDependencyLock.tag=mytag <tasks> commitLock
    
*dependencyLock.lockAfterEvaluating*

By default this option is true meaning that the locking will be applied in an `afterEvaluate` block. In certain circumstances
it can be advantageous to apply the lock outside of an `afterEvaluate` block; for example projects that explicitly trigger 
dependency configuration resolution via the `resolvedConfiguration()` method prior to when the lock would normally be 
applied in an `afterEvaluate` block. Thus, to lock outside of an `afterEvaluate` block, set the `lockAfterEvaluating`
project property to false as follows:

    ./gradlew -PdependencyLock.lockAfterEvaluating=false <tasks>

## Lock File Format

The lock file is written in a Json format. The keys of the map are made up of "\<group\>:\<artifact\>". The requested entry is informational to let users know what version or range of versions was initially asked for. The locked entry is the version of the dependency the plugin will lock to.

    {
        "configuration0": {
            "<group0>:<artifact0>": { "locked": "<version0>", "requested": "<requestedVersion0>" },
            "<group1>:<artifact1>": { "locked": "<version1>", "requested": "<requestedVersion1>" }      
        }
    }

If a dependency version selection was influenced by a command line argument we add a viaOverride field. The viaOverride field is informational.

    {
        "configuration0": {
            "<group0>:<artifact0>": { "locked": "<version0>", "requested": "<requestedVersion0>", "viaOverride": "<overrideVersion0>" }
        }
      
    }

If we include transitive dependencies.

    {
        "configuration0": {
            "<directgroup>:<directartifact>": { "locked": "<directversion>", "requested": "<directrequested>" },
            "<group>:<artifact>": { "locked": "<version>", "transitive": [ "<directgroup>:<directartifact>" ]}
        }
    }

If we don't include all transitive dependencies we still need to include the transitive information from the direct dependencies of other projects in our multi-project which we depend on. 

    {
        "configuration0": {
            "<directgroup>:<directartifact>": { "locked": "<directversion>", "requested": "<directrequested>" },
            "<group>:<artifact>": { "locked": "<version>", "firstLevelTransitive": [ "<mygroup>:<mypeer>" ]},
            "<mygroup>:<mypeer>": { "project": true }        
        }
    }

And we document project dependencies.

If you have

    ...
    dependencies {
      compile project(':common')
      ...
    }

The lock will have

    {
      "group:common": { "project": true }
    }
    
    
## Override File Format

Override files are simple Json constructs with keys representing groupId:artifactId and values for requested versions.  

    {
      "<group0>:<artifact0>": "version0",
      "<group1>:<artifact1>": "version1" 
    }


## Example

*build.gradle*

    buildscript {
        repositories {
            mavenLocal()
            jcenter()
        }
    
        dependencies {
            classpath 'com.netflix.nebula:gradle-dependency-lock-plugin:4.+'
        }
    }
    
    apply plugin: 'java'
    apply plugin: 'nebula.dependency-lock'
    
    repositories {
        mavenLocal()
        jcenter()
    }
    
    dependencies {
        compile 'com.google.guava:guava:14.+'
        testCompile 'junit:junit:4.+'
    }

    
When you run

    ./gradlew generateLock saveLock

It will output

*dependencies.lock*

    {
       "compile": {
           "com.google.guava:guava": {
               "locked": "14.0.1",
               "requested": "14.+"
           }
       },
       "default": {
           "com.google.guava:guava": {
               "locked": "14.0.1",
               "requested": "14.+"
           }
       },
       "runtime": {
           "com.google.guava:guava": {
               "locked": "14.0.1",
               "requested": "14.+"
           }
       },
       "testCompile": {
           "junit:junit": {
               "locked": "4.12",
               "requested": "4.+"
           },
           "com.google.guava:guava": {
               "locked": "14.0.1",
               "requested": "14.+"
           }
       },
       "testRuntime": {
           "junit:junit": {
               "locked": "4.12",
               "requested": "4.+"
           },
           "com.google.guava:guava": {
               "locked": "14.0.1",
               "requested": "14.+"
           }
       }
   }

# Possible Future Changes

### Determine Version Requested for Locked Transitives, Output Format

    {
      ...
      "<transitivegroup>:<transitiveartifact>": { "locked": "<transitiveLockedVersion>", "transitive": { "<group>:<artifact>": "<requestedVersion>", "<group1>:<artifact1>": "<requestedVersion1>" } }
      ...
    }

Gradle Compatibility Tested
---------------------------

Built with Oracle JDK7
Tested with Oracle JDK8

| Gradle Version | Works |
| :------------: | :---: |
| 2.2.1          | yes   |
| 2.3            | yes   |
| 2.4            | yes   |
| 2.5            | yes   |
| 2.6            | yes   |
| 2.7            | yes   |
| 2.8            | yes   |
| 2.9            | yes   |
| 2.10           | yes   |

LICENSE
=======

Copyright 2014-2016 Netflix, Inc.

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

<http://www.apache.org/licenses/LICENSE-2.0>

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
