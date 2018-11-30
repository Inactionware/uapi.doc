Development Notes
=================

[[toc]]

## Code generation

* The code generation template file name/path should avoid repetition, because there is only on class loader to load all template file, so if there are more than one template file has same file name or path in the different jar will cause load wrong template file.

## How to release version

* Update all project version in the configuration repo.
* Create new tag of the configuration repo, the tag name should be based on previously tag name.
* In project repo, update varialbe in setup-env.sh to point new tag of configuration repo.
* Create new tag or branch in project repo.

## Copyright text

    Copyright (c) $today.year. The UAPI Authors
    You may not use this file except in compliance with the License.
    You may obtain a copy of the License at the LICENSE file.

    You must gained the permission from the authors if you want to
    use the project into a commercial product.