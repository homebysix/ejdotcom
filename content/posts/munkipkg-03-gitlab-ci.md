---
title: "Building MunkiPkg packages using GitLab CI/CD"
date: 2021-11-07T22:55:00-08:00
slug: munkipkg-03-gitlab-ci
description: A method for automatically building installer packages from source using MunkiPkg and GitLab.
tags:
    - munkipkg
    - packaging
    - macadmin
    - gitlab
    - git
    - cicd
    - automation
---

In previous posts, I've [introduced you to MunkiPkg](../munkipkg-01-intro) and [provided some Git collaboration tips for MunkiPkg projects](../munkipkg-02-git). This time, I'd like to share a workflow for using GitLab's continuous integration (CI) features to build installers for MunkiPkg projects automatically. If your organization uses MunkiPkg for lightweight packaging and GitLab for source control, this automation can save you a great deal of time.

## Requirements

To follow along, you'll need the following:

- __GitLab__: Either the enterprise version or an account on [gitlab.com](https://about.gitlab.com/) will work.

- __Runner Mac__: A Mac that you can dedicate to the role of running GitLab tasks. The Mac should be physically secure — in a server rack or locked room, for example.

## Configure GitLab runner Mac

First, you'll configure your runner Mac with the necessary settings and software to run GitLab CI jobs.

1. Log in as a local administrator on your designated runner Mac.

1. Configure the Mac to never sleep. You can do this via **System Preferences → Energy Saver**, or with this Terminal command:

    ```
    sudo pmset -a autorestart 1 sleep 0 hibernatemode 0 disablesleep 1
    ```

    You can optionally configure a startup/wake schedule as a redundant way to ensure the Mac remains accessible:

    ```
    sudo pmset repeat wakeorpoweron MTWRFSU 06:00:00
    ```

1. In **System Preferences → Sharing**, Enable whatever type of remote access your team needs in order to administer this Mac and periodically install updates — for example, screen sharing (ARD/VNC) or remote login (SSH).

1. Create a non-admin user named __GitLab Runner__, username __gitlabrunner__. Choose a non-trivial password and store the password somewhere secure for your team.

    {{< admonition note "Why use a non-admin account?" >}}
    GitLab jobs executed by this runner will have shell access to your runner Mac. This is why you should avoid using the runner Mac for other work and minimize the privileges granted to the user that will be running the jobs. For more information, see the [Runner Security page of GitLab's documentation](https://docs.gitlab.com/runner/security/index.html).
    {{< /admonition >}}

1. Optional: Install [Homebrew](https://brew.sh). This can simplify some of the subsequent installs.

1. Remain logged in as an administrator, and follow the [macOS GitLab runner binary installation instructions](https://docs.gitlab.com/runner/install/osx.html). (You can use the __Homebrew installation__ instructions if Homebrew is installed on the Mac. Otherwise use the __Manual installation__ instructions.)

1. Install Git (via `brew` or [download here](https://git-scm.com/download/mac)).

1. Install Python 3 (via `brew` or [download here](https://www.python.org/downloads/macos/)).

1. Install the PyYAML module for Python 3:

    ```
    pip3 install --upgrade PyYAML
    ```

    (This allows using MunkiPkg with `build-info.yaml` metadata files.)

1. Create a local clone of the MunkiPkg project:

    ```
    git clone https://github.com/munki/munki-pkg /Users/Shared/munki-pkg
    ```

1. Add `munkipkg` to the shell path using a symbolic link:

    ```
    ln -s /Users/Shared/munki-pkg/munkipkg /usr/local/bin/munkipkg
    ```

1. Optional: Configure automatic login for the **gitlabrunner** user account in **System Preferences → Users & Groups**.

1. On the runner Mac, log out as the administrator account, and log in as **gitlabrunner**.

1. Go to your **Settings** page on GitLab (either main settings or the settings of a group you're a member of), and click the **CI/CD** settings page. Then locate the **Runners** section and click **Expand**, and you'll see an area called **Specific Runners**. Make a note of the URL and registration token shown there.

1. On the runner Mac, follow the [runner registration instructions](https://docs.gitlab.com/runner/register/index.html#macos), providing the following information when prompted:

    - __URL__: (paste the URL displayed on the group CI/CD settings page above)
    - __Token__: (paste the token displayed on the group CI/CD settings page above)
    - __Description__: (press Return to accept the default hostname, or type your own)
    - __Tags__: `munkipkg`
    - __Executor__: `shell`

1. Return to your group CI/CD settings page on GitLab and verify that the new Mac runner appears in the list of group runners.

1. On the runner Mac (logged in as the **gitlabrunner** account), open Terminal and start the GitLab Runner service:

    ```
    gitlab-runner install && gitlab-runner start
    ```

1. Verify that the GitLab runner is working:

    ```
    gitlab-runner status
    ```

    The output should include `gitlab-runner: Service is running`. If not, pause here and troubleshoot.

    {{< admonition tip "Check log permissions" >}}
    If your runner stops immediately after starting, check that the **gitlabrunner** user has write access to the `StandardOutPath` and `StandardErrPath` locations specified in the GitLab runner LaunchAgent.
    {{< /admonition >}}

1. Restart the runner Mac. (And log in as **gitlabrunner** if needed.) Run `gitlab-runner status` again to verify the runner is working.

## Configure build script

The next part of the process creates the CI configuration file and a script that triggers MunkiPkg builds.

{{< admonition note "Adjust paths if needed" >}}
The script and CI configuration below assume that your MunkiPkg project folders are all in the root level of your repository. If you put your MunkiPkg projects in a subfolder, you'll need to add another layer to the paths in the `for` loop in the shell script and the `artifacts:paths` in the yaml config.
{{< /admonition >}}

1. In the repository where you store MunkiPkg project sources, create a file called `munkipkg_build.sh` with the following contents:

    ```sh {linenos=table}
    #!/bin/sh
    # This script is referenced by .gitlab-ci.yml and uses
    # MunkiPkg to build projects in this repository.
    MUNKIPKG="/usr/local/bin/munkipkg"
    for proj in */build-info.*; do
        python3 $MUNKIPKG "$(dirname "$proj")" || exit 1
    done
    ```

1. Also create a file called `.gitlab-ci.yml` with the following contents:

    ```yaml {linenos=table}
    munkipkg_build:
      script:
        - sh munkipkg_build.sh
      artifacts:
        paths:
          - "*/build/*.pkg"
      tags:
        - munkipkg
      only:
        - main
    ```

1. Commit and push the `munkipkg_build.sh` and `.gitlab-ci.yml` files to GitLab.

## Try it out

Now the fun part: any time you push a commit to the `main` branch of the repository, MunkiPkg will automatically build packages and upload them as artifacts to GitLab.

1. On your regular Mac (not your GitLab runner), create a demonstration MunkiPkg project:

```
munkipkg --create munki_kickstart
touch munki_kickstart/payload/Users/Shared/.com.googlecode.munki.checkandinstallatstartup
```

1. Commit and push the new project to GitLab.

1. Navigate to your project on GitLab and click the **CI/CD** section in the left sidebar. Click the **Jobs** sub-section.

1. There should be a small "download" icon for the most recent job. Click the icon to download the artifacts zip file.

    ![Download Artifact Icon](../images/munkipkg-03-gitlab-ci-download.png)

1. Unzip the artifacts file and you'll see a package for each of your MunkiPkg projects.

    ![Artifact Contents](../images/munkipkg-03-gitlab-ci-unzip.png)

## Example repository

I've created a working example repository on GitLab that you can use as a reference:

[https://gitlab.com/elliotjordan/munkipkg-cicd-build](https://gitlab.com/elliotjordan/munkipkg-cicd-build)

## Next steps

{{< admonition note "Runner maintenance" >}}
Routine care and feeding of your GitLab runner Mac might include performing the following tasks as a local administrator:

- Install macOS and Xcode command line tools updates
- If Homebrew is installed: `brew update`
- PyYAML upgrades: `pip3 install --upgrade pip PyYAML`
- Git syncs: `cd /Users/Shared/munki-pkg; git fetch; git pull`
- [GitLab runner updates](https://docs.gitlab.com/runner/install/osx.html#manual-update)
{{< /admonition >}}

Now that you've got automation building your packages, you may want to consider creating additional automation that downloads your packages and imports them into your software distribution tool (Munki or Jamf, for example). One way to do this would be to create an override for my [GitLabArtifact family of AutoPkg recipes](https://github.com/autopkg/homebysix-recipes/tree/master/GitLabArtifact). I'll go into more detail on this in a future post.
