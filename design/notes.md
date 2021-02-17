Development Notes
=================

## How to start new feature/issue development?

* In `project repo`, create a new branch for new feature development project, the branch name likes "feature/xxx" or "bugfix/xxx".
* In `configuration repo`, create new branch which name should likes "uapi/feature/xxx" or "uapi/bugfix/xxx.
* In `configuration repo`, increse developed project version number.
* In `project repo`, modify configuration item (cfgBranch=xxx) in your developed projects build script (setup-env.sh) to point new version which is defined in `configuration repo`.
* Start development.

## How to create new release? (Out of Date)
* In `configuration repo`, create new git branch named likes "uapi/vxx, the `xx` is version number.
* In `configuration repo`, change project's type to `stable` if necessary.
* In `project repo`, create new "release branch", the branch pattern is "release/v[major verion]-[minor verion]-[fix version]", minor version and fix version is optional.
* In `project repo`, update config item (cfgBranch=xxx) in projects build script (setup-env.sh) in release branch to point new git branch in `configuration repo`.

## How to fix issue on released project? (Out of Data)

* Create a new branch based on git "release branch" in project repo, the name likes "bugfix/xxx".
* Create new branch based on git tagged "Configuration" project, the branch name likes "bugfix/xxx".
* Increse version number of projects which is your developing in "Configuration" project.
* Modify configuration in (cfgBranch=xxx) in your projects build script (setup-env.sh) to point to new Configuration project branch.
* When you finish development, you should create PR to "release branch".
* Create new git tag for "Configuration" project based on the branch which is created on step 2.
* Modify configuration in (cfgBranch=xxx) in your projects build script (setup-env.sh) to point to new tag in "Configuration repo".
* Remove "bugfix/xxx" branch on both "project repo" and "configuration repo". 

## VM option for application which require Netty

Upgrade to java 9 and above version, Netty will throw ClassNotFound exception for sun.misc.UnSafe, add below statement in module.info file:
```java
requires jdk.unsupported;
```

Upgrade to java 9 and above version, Netty will throw some IllegalAccessException, add below in vm arguments:
```java
--add-opens java.base/jdk.internal.misc=io.netty.all
--add-opens java.base/java.nio=io.netty.all
-Dio.netty.tryReflectionSetAccessible=true
```

## Copyright text

Copyright (c) $today.year. The UAPI Authors
You may not use this file except in compliance with the License.
You may obtain a copy of the License at the LICENSE file.

You must gained the permission from the authors if you want to
use the project into a commercial product.