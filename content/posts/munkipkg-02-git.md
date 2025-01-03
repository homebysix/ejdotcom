---
title: "Collaborating on MunkiPkg projects with Git"
date: 2021-10-04T08:00:00-07:00
slug: munkipkg-02-git
description: Recommendations and considerations for collaborating on MunkiPkg projects in a Git repo.
tags:
    - munkipkg
    - packaging
    - macadmin
    - git
---

[MunkiPkg](https://github.com/munki/munki-pkg) is a fantastic way to build lightweight packages (see my [earlier post](../munkipkg-01-intro) to get started), but it really stands out when you collaborate with others to maintain package sources in a Git repository.

As with any collaborative effort, it helps to establish some shared guidelines and conventions. Here are a few I've successfully used when building MunkiPkg projects with a team.

## Read Me

The inclusion of a read me file is essential, not just for others who may not be familiar with MunkiPkg (although that's important), but also for yourself a year or two in the future. Being able to document your package's contents concisely bodes well for the package's overall quality.

Here's a sample `README.md` (in Markdown format) I often use as a starting point:

```md
# Outset - Dock Script

The installer produced by this source project installs an
[Outset](https://github.com/chilcote/outset) script that sets the standard Dock
items at user login.

This folder is a [MunkiPkg](https://github.com/munki/munki-pkg) project. After
making changes, be sure to increment the version in build-info.yaml, then build
the project with the `munkipkg` tool. The resulting pkg file will appear in the
build folder.
```

Be sure to mention in the read me if there are additional requirements — for example, the presence of a signing certificate in the keychain.

## Semantic Versioning

More so than some other package creation tools, MunkiPkg firmly encourages you to do the "right" things when it comes to versioning. Its sane defaults include:

- The version number is *not* in the bundle identifier of the package
- The package filename is `NAME-VERSION.pkg` so you can have multiple versions of the same package together in a folder
- The `version` metadata starts at 1.0, with the understanding that future versions always increase

On top of these default behaviors, you should consider using [semver](https://semver.org) for your versioning convention, for the benefit of two audiences:

- __Humans__: Well constructed versions have meanings that are obvious even to those who didn't design the package.
- __Computers__: Good software update frameworks can compare the package versions to the installed receipts in order to determine whether a Mac is eligible for an update to your package.

## Package Granularity

If you're experienced in Mac systems administration, you probably already know that it's considered a good practice to install app settings or configurations in a separate package from the app itself. That way when the app gets an update (or the settings change) you don't need to repackage everything. (For details, see [*Packaging for Apple Administrators*](https://scriptingosx.com/packaging-for-apple-administrators/) by Armin Briegel; chapter 6, section 3.)

MunkiPkg excels at creating these types of lightweight, purpose-specific configuration installers, especially if the configuration file(s) to be deployed is in common text-based formats like plist or JSON.

One recurring task many Mac admins use MunkiPkg for is the creation of installers for [Outset](https://github.com/chilcote/outset) scripts, and the above principle applies there too. Deploying an installer for your Outset script(s) separate from the Outset installer itself makes it easier to update each in the future.

(For an example of a MunkiPkg project for a docklib script, see [this post](../docklib-outset/).)

## Be Git-Friendly

Some things that MunkiPkg can package aren't necessarily the kind of things you'd want to put into Git. For example, packages that contain private certificates, licensing information, credentials, or other types of secrets should generally not be stored on Git or any other version control system. One solution is to use the _.gitignore_ file to exclude secrets from the project, but then you have to figure out how to shuttle the secrets around if there are multiple package contributors. It may be better to find a more secrecy-minded location to store these package projects.

<!-- TODO: GitLab secrets? -->

Likewise, projects that contain a large number of binary (non-text) files might not be appropriate for collaboration on Git, since Git cannot track meaningful changes to binary files. This would include apps, plugins, and other macOS bundles, large images, and compiled executables. You can still use MunkiPkg to create installers that contain these files, but you may not benefit as much from putting those MunkiPkg projects into Git.

## Pre-Commit

I've created a [set of hooks for the pre-commit framework](https://github.com/homebysix/pre-commit-macadmin) that will help catch common errors in MunkiPkg projects. After [setting up pre-commit](../pre-commit-01-intro), create a file called _.pre-commit-config.yaml_ at the root of the Git repo you store MunkiPkg projects in, and add these contents:

```yaml {linenos=table}
repos:
- repo: https://github.com/homebysix/pre-commit-macadmin
  rev: v1.17.0
  hooks:
  - id: check-munkipkg-buildinfo
  - id: check-outset-scripts
  - id: check-plists
```

Then navigate to the repo using `cd` and run `pre-commit install` to activate the hooks.

My hooks will alert you and stop a commit from happening if any of these errors are detected:

- Build-info (plist, JSON, or YAML) file is missing or incorrectly formatted
- Outset scripts are not set to executable
- Any plist (including in the payload) is incorrectly formatted

{{< admonition note "Note" >}}
My pre-commit hooks do not verify whether <em>./scripts/preinstall</em> or <em>./scripts/postinstall</em> files are executable, because running <code>munkipkg</code> to build the project <a href="https://github.com/munki/munki-pkg/blob/71d270833719881b0f9f6002a476e716bc232505/munkipkg#L911-L918" target="_blank">will set those files as executable</a> automatically.
{{< /admonition >}}

If your whole team uses the pre-commit configuration above, the chances of these specific errors creeping into your package source decreases to nearly zero.

## Sharing is Caring

The foundation of the Mac admin community is sharing solutions. If you've used MunkiPkg to solve a problem in a way that you're particularly proud of, and if you believe others may benefit in knowing the solution, share it!

Many examples of projects that use MunkiPkg live on GitHub, including some great tools like [Nudge](https://github.com/macadmins/nudge), [UMAD](https://github.com/macadmins/umad), [InstallApplications](https://github.com/macadmins/installapplications), [JSSImporter](https://github.com/jssimporter/JSSImporter), and [jss_helper](https://github.com/jssimporter/jss_helper). Also see the [munki-pkg-projects](https://github.com/munki/munki-pkg-projects) repository for some older examples.
