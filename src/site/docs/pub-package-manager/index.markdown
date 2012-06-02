---
layout: default
title: "pub: The Dart Package Manager"
description: "With pub, you can use third-party Dart libraries in 
your web and command-line Dart apps."
---

# {{ page.title }}

<aside class="note">
  <b>Note:</b> pub is under active development, so expect
  some changes to the functionality of the tool and these docs.
</aside>

This page tells you how to use the _pub_ tool (`bin/pub`)
to manage Dart packages. A Dart package is simply
a directory containing any number of Dart libraries and their dependencies.

The `bin/pub` executable is in the [Dart SDK](/docs/sdk/).
You can either [download the SDK separately](/docs/sdk/#download)
or get it as part of the [Dart Editor package](/docs/editor/#download).

## Creating a pubspec file

A **pubspec** file defines an application's package dependencies, and
where to get the packages. Put this file in the same directory as the
file containing the app's 
main() method.

Here is an example of a pubspec file that associates the
`awesome` package with a git repository:

{% pretty_code console 0 %}
dependencies:
  awesome:
    git: git://github.com/munificent/awesome.git
{% endpretty_code %}

A pubspec file uses the [YAML](http://yaml.org/) format.

## Installing the packages

Now that you have a pubspec file, you can run `pub install`. You must
run this command from the directory containing the pubspec file.

{% pretty_code console 0 %}
cd your/app
$DART_SDK/bin/pub install
{% endpretty_code %}

This creates a _packages_ directory inside the current directory,
and clones the git repositories into the packages directory.

The packages directory also contains any _transitive
dependencies_. 
For example, if the `awesome` package is dependent on the `sweet` package, 
the pub tool will also download and install the `sweet` package.

## Pointing Dart to your packages

The Dart runtime needs to know where the packages are located.
You can specify where packages are located for both the Dart Editor
and the Dart VM.

If you use the `pub install` command, the packages directory is created
for you next to the pubspec file.

### Mapping packages for the Dart Editor

To help the Dart Editor understand how to find packages, go to Preferences 
and point the "Package directory" to `/path/to/your/app/packages`.

<aside class="note">
  <b>Note:</b>  In the future, you'll be able to configure the packages
  mapping per project, instead of globally.
</aside>

### Mapping packages for the Dart VM

For command-line Dart applications that use packages, use the
`--package-root` option to specify the packages location.

For example:

{% pc console 0 %}
$DART_SDK/bin/dart --package-root=./packages/ app.dart
{% endpc %}

Note the required trailing slash when specifying the `./packages/` directory.

## Importing libraries from packages

To import libraries found in packages, use the `package:` prefix.

{% pretty_code dart 0 %}
#import('package:awesome/awesome.dart');
{% endpretty_code %}

## Additional options

Run `$DART_SDK/bin/pub --help` for a list of commands.

{% include syntax-highlighting.html %}