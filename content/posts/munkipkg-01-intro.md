---
title: "You Might Like MunkiPkg"
date: 2021-01-24T17:00:00-08:00
slug: munkipkg-01-intro
description: Introduction to MunkiPkg, an overview of its best suited use cases, and a simple walkthrough for your first project.
tags:
    - munkipkg
    - packaging
    - macadmin
---

The Mac admin community benefits from a wealth of open-source tools for solving our shared challenges. I'd like to highlight one such tool that I've come to depend on: [MunkiPkg](https://github.com/munki/munki-pkg). Created by the prolific Greg Neagle, MunkiPkg streamlines the process of building Mac installer packages from source files.

IT and security professionals have needed to install software and other items on their Mac fleets for decades now, so of course there's no shortage of packaging tools available. Projects like [The Luggage](https://github.com/unixorn/luggage), [pkggen](https://github.com/korylprince/pkggen), [PackageMaker](https://developer.apple.com/library/archive/documentation/DeveloperTools/Conceptual/PackageMakerUserGuide/Introduction/Introduction.html), [Packages](http://s.sudre.free.fr/Software/Packages/about.html), [Composer](https://www.jamf.com/products/jamf-composer/), and [QuickPkg](https://github.com/scriptingosx/quickpkg) all serve this need with varying degrees of simplicity and automation. Even with all those choices available, I often find myself reaching for MunkiPkg when creating a new custom package for a few compelling reasons:

- **Collaboration-friendly.** The ability to check MunkiPkg projects into a Git repository and share maintenance duties with colleagues gives MunkiPkg an edge over tools like Composer that are built to store sources on a single device.

- **Human-readable config.** The package metadata for MunkiPkg projects is in plist format by default, but can be in JSON or YAML if desired. I prefer YAML due to its brevity and readability.

- **Automatable.** If your Git host supports CI/CD workflows, it's a straightforward process to configure your runner to build packages from source when the main branch is updated.

- **Simple, but capable of handling complexity.** A generic MunkiPkg project contains only a few simple ingredients (payload, scripts, and metadata). If you require more advanced permissions and configurations, a Bom.txt file and additional metadata keys are available.

- **Sane defaults.** MunkiPkg's default settings encourage consistent package receipt identifiers and [semantic versioning](https://semver.org/), and projects are structured in a way that is both understandable and non-proprietary.

{{< admonition tip "A note about the name" >}}
Don't be fooled: MunkiPkg does <em>not</em> require or depend upon <a href="https://www.munki.org/munki/">Munki</a>. MunkiPkg can be used to create installer packages that work with any software deployment tool that deploys .pkg files, including Munki, <a href="https://www.jamf.com/products/jamf-pro/">Jamf Pro</a>, and many others.
{{< /admonition >}}

## Walkthrough

To demonstrate MunkiPkg's simplicity, I'd like to lead you through creation of an installer that places your company logo at a predetermined location on disk.

### Installing MunkiPkg

MunkiPkg is a Python script, so my preferred way of "installing" it is to clone the GitHub repository and place a symbolic link (symlink) in the shell `PATH`.

Cloning the Git repo instead of just downloading the script itself ensures that you can use `git pull` to get the latest changes when MunkiPkg is updated in the future.

{{< admonition note "note" >}}
    The steps below use <em>~/Developer</em> to store your sources, but feel free to change to <em>~/src</em>, <em>~/Documents</em>, or wherever you store your source code.
{{< /admonition >}}

1. Clone the MunkiPkg project from GitHub.

        git clone https://github.com/munki/munki-pkg.git ~/Developer/munki-pkg

2. Create a folder at _/usr/local/bin_ if it doesn't already exist:

        mkdir -p /usr/local/bin

3. Create a symlink in _/usr/local/bin_ that points back to the _munkipkg_ script itself.

        ln -s "$HOME/Developer/munki-pkg/munkipkg" /usr/local/bin/munkipkg

4. Test it out.

        munkipkg --version

    If successful, you should see the current MunkiPkg version number in the output.

### Creating a project

Now that MunkiPkg is installed, you use the `--create` flag to create a new project:

    munkipkg --create ~/Developer/pretendco_logo

Or if you prefer a format other than plist for your build-info files, you can specify JSON:

    munkipkg --create --json ~/Developer/pretendco_logo

Or YAML (my personal preference):

    munkipkg --create --yaml ~/Developer/pretendco_logo

(Install PyYAML using `pip install -U PyYAML` first, if needed.)

This will create a folder at the specified path with the necessary scaffolding. You'll see the following:

- __build-info.plist__ (or __.yaml__ or __.json__)

    The metadata for the package, including identifier and version.

- __payload__

    The folder that contains the paths that will be installed on the target volume. Empty by default.

- __scripts__

    The folder that contains preinstall or postinstall scripts. Empty by default.

### Customizing your project

Once you've created the project, you'll want to customize the following:

1. Open the _build-info_ plist, yaml, or json file and adjust as desired:
    - The `name` will be the filename of the resulting package.
    - The `identifier` will be used when creating a package receipt. It's conventional to use reverse domain naming based on a domain you or your organization owns, like `com.pretendco.logo`.
    - The `version` is `1.0` by default. You'll increment this upward as you modify and rebuild the package.

2. Copy the logo into the desired path within the _payload_ folder. Any folder structure you create within _payload_ will be recreated on the target disk when you run the resulting installer.

        cd ~/Developer/pretendco_logo
        mkdir -p payload/Library/PretendCo/
        cp ~/Downloads/logo.png payload/Library/PretendCo/logo.png

3. Since this example project doesn't require any preinstall or postinstall scripts, you can remove the scripts folder or just ignore it.

        rm -r scripts/

    At this point your project will look like this:

    ![MunkiPkg PretendCo Logo Example](../images/munkipkg-01-intro.png)

### Building the package

Once the project metadata and contents are ready, you can build the package by running `munkipkg` followed by the path to the project.

    munkipkg ~/Developer/pretendco_logo

Note that you can just use `.` if your current working directory is the project folder itself.

    cd ~/Developer/pretendco_logo
    munkipkg .

MunkiPkg supports [idempotency](https://www.infoworld.com/article/3263724/idempotence-and-the-discipline-of-devops.html), so building the project repeatedly from the same source should produce exactly the same result.

Once built, the finished package will appear in the _build_ folder inside your project. From here, you can copy the package to your software distribution tool and send it to your company's Macs as you normally would.

### Modifying the package

Let's say your company decides to change their logo. No need to start over! Instead, just tweak the project like so:

1. Replace the _logo.png_ file in the _payload_ folder.
2. Increment the version in the _build-info_ file (for example, from `1.0` to `1.1`).
3. Build the project and import the resulting package into your software distribution tool.

MunkiPkg really shines here by encouraging you to treat package sources the same way you would treat a software code base: improve, increment, build, release.

## When to use other tools

It's rare that a single tool will satisfy all possible scenarios, and MunkiPkg is no exception. Here are a few specific use cases that may favor other packaging tools:

- **Publicly-downloadable software.** In my opinion, [AutoPkg](https://github.com/autopkg/autopkg) is by far the best way to package and process Mac software that can be downloaded from a public URL.

- **Applications or other software bundles.** If you need to manually wrap an app or other bundle into a software installer and can't use AutoPkg, you may want to check out [QuickPkg](https://github.com/scriptingosx/quickpkg).

- **Installers that require interface customization.** The installers that MunkiPkg produces are ideal for silent background installation, but if you require a custom installer interface or installation choices, a more full-featured tool like [Packages](http://s.sudre.free.fr/Software/Packages/about.html) may be what you need.

## Further Reading

I'll be returning to the topic of MunkiPkg in the future. Until then, here are a few articles and resources about MunkiPkg:

- Greg Neagle: [Introducing MunkiPkg](https://managingosx.wordpress.com/2015/07/28/introducing-munkipkg/)
- REDline: [Using MunkiPkg](https://redlinetech.wordpress.com/2015/07/31/using-munki-pkg/)
- Mat X: [Use Munki to install a screensaver](https://macvfx.blog/2019/04/27/use-munki-to-install-a-screensaver/)
- Josh Hovinetz: [Signing Package Bundle Installers With munkipkg](https://sincerelyjoshin.github.io/signing_package_bundle_installers_with_munkipkg.html)
- Alan Siu: [Don’t store your munkipkg files in Google Drive File Stream](https://technology.siprep.org/dont-store-your-munkipkg-files-in-google-drive-file-stream/)

Armin Briegel's book _[Packaging for Apple Administrators](https://scriptingosx.com/packaging-for-apple-administrators/)_ is an excellent resource and dedicates a few pages to MunkiPkg.

For near-realtime help, the people in the [#packaging channel in MacAdmins Slack](https://macadmins.slack.com/messages/C0EGLTFR7) can answer questions about MunkiPkg and packaging in general.
