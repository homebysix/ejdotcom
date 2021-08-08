---
title: "Simplifying pkg recipes with AppPkgCreator"
date: 2021-08-07T22:00:00-07:00
tags:
    - autopkg
    - python
    - macadmin
---

Before AutoPkg's 1.0 release in November 2016, it took at least three processors to create a simple pkg recipe that installs an app into _/Applications_: [PkgRootCreator](https://github.com/autopkg/autopkg/wiki/Processor-PkgRootCreator) to create the folder structure, [Copier](https://github.com/autopkg/autopkg/wiki/Processor-Copier) to copy the app into place, and [PkgCreator](https://github.com/autopkg/autopkg/wiki/Processor-PkgCreator) to create the actual installer package.

With the release of AutoPkg 1.0, the [AppPkgCreator](https://github.com/autopkg/autopkg/wiki/Processor-AppPkgCreator) processor allowed simplifying those steps into one for many recipes. Not only does this make a large number of AutoPkg recipes much simpler and easier for administrators to understand, but it also streamlines `autopkg audit` checks.

## PkgCreator to AppPkgCreator script

To help encourage administrators to use the AppPkgCreator processor, I've written a Python script that does all the work of converting recipes for you.

Here's the source of the script; see below for usage information and tips.

{{< gist homebysix 0c42be278696717f28a56c819fc4fddb >}}

## Usage

As with many command line tools, running `/path/to/PkgCreator_to_AppPkgCreator.py --help` will produce usage information.

```
% ./PkgCreator_to_AppPkgCreator.py --help
usage: PkgCreator_to_AppPkgCreator.py [-h] [--auto] [-v] repo_path

PkgCreator_to_AppPkgCreator.py

Script for converting compatible AutoPkg "pkg" type recipes from using
PkgRootCreator-Copier-PkgCreator to using AppPkgCreator.

positional arguments:
  repo_path      Path to search for AutoPkg recipes.

optional arguments:
  -h, --help     show this help message and exit
  --auto         Automatically apply suggested changes to recipes. Only
                 recommended for repos you manage or are submitting a pull
                 request to. (Applying changes to repos added using `autopkg
                 repo-add` may result in failure to update the repo in the
                 future.)
  -v, --verbose  Print additional output useful for troubleshooting. (Can be
                 specified multiple times.)
```

## Automatically apply changes

To automatically update eligible recipes in your AutoPkg recipe repository to use the AppPkgCreator processor, run this command (substituting the appropriate paths):

```sh
/path/to/PkgCreator_to_AppPkgCreator.py --auto /path/to/your-recipes
```

You'll see output that looks like this:

```
✨ Recipe ./Foo/FooPro.pkg.recipe is eligible for AppPkgCreator. Converting...
✨ Recipe ./Bar/BarApp.pkg.recipe is eligible for AppPkgCreator. Converting...
✨ Recipe ./Baz/BazSuite.pkg.recipe is eligible for AppPkgCreator. Converting...
```

Review the changes made by the script, and run each modified recipe to make sure it works as expected. Once the modified recipes have been tested successfully, you can commit and push the changes.

## Reverting

If a converted recipe doesn't work for any reason, you can revert the changes using Git:

```sh
cd /path/to/your-recipes
git checkout -- path/to/recipe
```

## Marking Exemptions

You might have reasons not to switch to AppPkgCreator in certain recipes. For example, if you think a recipe may require adding a preinstall/postinstall script in the future.

To ignore specific recipes when running this script, add the phrase `Cannot use AppPkgCreator` somewhere in the Comment or Description of the recipe. ([Example here](https://github.com/autopkg/homebysix-recipes/blob/0de744cca186faa592ea1ef891872deff50ce94e/ShirtPocket/SuperDuper.pkg.recipe#L5-L6).)

## Manual Changes

In many cases, you may also be able to remove PlistReader, Versioner, or AppDmgVersioner processors, since AppPkgCreator itself includes versioning (using the `CFBundleShortVersionString` key). These changes are not done automatically by the script, so you'll need to make them manually if needed.

## Thank you!

I periodically review AutoPkg recipes written by others, and I'm always grateful when a recipe author finds a way to simplify and streamline a recipe I use. Often, those improvements inspire me to write bulk automation like this, which I hope will enable and encourage the positive cycle of community refinement to continue.
