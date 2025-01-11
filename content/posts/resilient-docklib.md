---
title: "Writing resilient docklib scripts"
date: 2021-09-04T17:00:00-07:00
description: Detailed guide for designing and developing a resilient Python script that uses docklib to modify the Dock on your Mac fleet.
slug: resilient-docklib
tags:
    - macadmin
    - docklib
    - python
---

Recently I needed to perform a time-honored ritual of the modern Mac client systems engineer: customizing the items that would appear in the macOS Dock when users first log in to a newly-provisioned Mac. Numerous examples and resources exist on the topic, and I'm the maintainer of a [framework](https://github.com/homebysix/docklib/) for performing the task. Should be a walk in the park, right?

*Wrong!*

I stumbled onto a couple interesting roadblocks, and I might not be the first or last to do so. To help others (and my future self) maximize the resilience and longevity of their own docklib scripts, I'll share how I overcame these roadblocks, and how I think about creating Dock scripts in general.

{{< admonition tip "Python knowledge helpful but not required" >}}
In this post I'll use [docklib](https://github.com/homebysix/docklib/) to write a Dock configuration script in Python. I'll also include comments that can help you follow along if you aren't familiar with Python. (I often find reading other people's scripts quite helpful when learning a language.)
{{< /admonition >}}

## Contents  <!-- omit in toc -->

I acknowledge that the full process of managing your fleet's Dock configuration includes some things that aren't covered by this post. For example, I won't go into details here on deploying Outset or a Python runtime, nor will I cover how to deploy your Dock script using Munki or Jamf. This post focuses on the design and resilience of your Dock script itself.

- [Define the goal](#define-the-goal)
- [Select a tool](#select-a-tool)
- [Create a test environment](#create-a-test-environment)
- [Specify Dock contents](#specify-dock-contents)
- [Skip missing apps](#skip-missing-apps)
- [Avoid monolingual assumptions](#avoid-monolingual-assumptions)
- [Define the scope and conditions](#define-the-scope-and-conditions)
- [Consider user context](#consider-user-context)
- [Enable idempotence](#enable-idempotence)
- [Prevent race conditions](#prevent-race-conditions)
- [Make backups](#make-backups)
- [Simplify troubleshooting](#simplify-troubleshooting)
- [Tying it all together](#tying-it-all-together)

## Define the goal

Before diving into coding a Dock script, one question should always be answered first: {{% mark %}}Will your management of Dock configurations provide a clear benefit to your users?{{% /mark %}} If the answer to this question is no, then stop here. (Congratulations, you've just saved yourself and your colleagues a fair bit of time.)

![Default macOS Dock](../images/default-dock.png)

Apple's selection of items included in default macOS Docks is far from unreasonable. All the basics are included: a good web browser, many productivity apps, media storage and entertainment, system settings, and avenues for finding more apps. When choosing to manage the Dock (or any other manageable setting), IT administrators should be confident they are improving the user experience, not simply codifying their own personal preferences.

The most persuasive arguments I've seen to justify Dock scripts are:

- minimizing new employee toil on day one by making it easy to find essential apps
- reducing confusion and support calls by gently discouraging use of unsupported built-in apps

Once you're confident that managing the Dock is the right way forward, then it's time to consider which tool you'll use.

## Select a tool

At its most basic, the Dock configuration file is just a property list (plist), and macOS contains several built-in tools meant to create and modify plist files (including but not limited to `plutil`, `defaults`, and `PlistBuddy`). However, the Dock plist in particular contains complex nested data structures that benefit from a layer of abstraction when handling programmatically. Open-source tools have been created to provide this layer of abstraction.

One such tool many Mac admins (including myself) have relied on for years is **[dockutil](https://github.com/kcrawford/dockutil)**, which provides shell-like commands for Dock management.

For Mac admins who do most of their scripting in Python, the **[docklib](https://github.com/homebysix/docklib)** module aims to be a familiar, flexible, and _Pythonic_ way to express the desired state of the Dock. For the purposes of this exercise I'll be using Python/docklib, but all of the concepts below could be converted for shell/dockutil.

It's also possible to manage the Dock configuration using [MDM profiles](https://developer.apple.com/documentation/devicemanagement/dock) or configuration management tools like [Chef](https://www.chef.io/), but I won't go into those methods here.

## Create a test environment

Before you start creating your script, it's a good idea to create an environment you can use to test incrementally and iterate your changes in isolation. While it's certainly possible to do this testing on your daily driver Mac, you may find that using a virtual machine or test Mac gives you extra confidence and allows you to replicate the first login experience more closely.

You'll need to ensure your test Mac has Python 3 and the docklib module installed. Numerous ways exist to meet this requirement, but my go-to advice for most admins is to install the [MacAdmins Python "recommended" package](https://github.com/macadmins/python/releases), which includes docklib. With this package installed on your test Mac, you can use the symlink at `/usr/local/bin/managed_python3` as your Python 3 interpreter.

Let's run through a rudimentary test of our ability to back up, make a Dock change, and restore.

1. Back up your test Mac's Dock configuration with this Terminal command:

    ```sh
    cp ~/Library/Preferences/com.apple.dock.plist /tmp/com.apple.dock.backup.plist
    ```

1. Create a bare-bones Python script with the following contents (adjust your interpreter path, if you aren't using MacAdmins Python). For convenience I suggest saving the file to `~/Desktop/dock_script.py`.

    ```py {linenos=table}
    #!/usr/local/bin/managed_python3
    from docklib import Dock

    dock = Dock()
    item = dock.makeDockAppEntry("/System/Applications/Chess.app")
    dock.items["persistent-apps"].append(item)
    dock.save()
    ```

1. Make the script executable:

    ```sh
    chmod +x ~/Desktop/dock_script.py
    ```

1. Run the script:

    ```sh
    ~/Desktop/dock_script.py
    ```

1. Verify the Chess app was successfully added to your Dock.

1. Revert the Dock configuration and relaunch the Dock:

    ```sh
    cp /tmp/com.apple.dock.backup.plist ~/Library/Preferences/com.apple.dock.plist
    killall cfprefsd Dock
    ```

1. Verify your Dock config is back to its original state, without the Chess app.

If that worked as expected, you can use steps 4 through 7 above to repeatedly test your Dock script as you iterate and improve it, restoring your Dock to its previous state each time.

Now you're ready to dive into the script itself.

## Specify Dock contents

The most obvious question: {{% mark %}}What items do you want in your users’ initial Dock?{{% /mark %}} Digging into your answer's details can reveal a surprising amount of nuance. How you define the Dock contents will determine how static or dynamic your script is and how future macOS changes are handled.

### Example: Static list of apps  <!-- omit in toc -->

On the static end of the spectrum, you can define a comprehensive list of the apps you want in the Dock (`desired_apps` below). The primary benefit of this method is predictability. You'll get the apps you specify, in the order you specify — and nothing else.

```py {linenos=table, hl_lines=["8-16",22], linenostart=1}
#!/usr/local/bin/managed_python3

from docklib import Dock

# Load current Dock
dock = Dock()

# Define list of apps, from left to right
desired_apps = [
    "/System/Applications/Launchpad.app",
    "/Applications/Google Chrome.app",
    "/Applications/Microsoft Outlook.app",
    "/Applications/Slack.app",
    "/Applications/Managed Software Center.app",
    "/System/Applications/System Preferences.app",
]

# Set persistent-apps as desired
p_apps = []
for app in desired_apps:
    p_apps.append(dock.makeDockAppEntry(app))
dock.items["persistent-apps"] = p_apps

# Save changes and relaunch Dock
dock.save()
```

Only `persistent-apps` is defined above, but you can define a list of `persistent-others` too if desired.

### Example: Dynamic adds, removes, and replacements  <!-- omit in toc -->

On the dynamic end of the spectrum would be a script that defines the granular _changes_ needed to bring the Dock to your desired config, rather than defining the config wholesale. The example below uses a dictionary (`app_changes`) to define the specific additions, removals, and replacements needed:

```py {linenos=table, hl_lines=["8-30"], linenostart=1}
#!/usr/local/bin/managed_python3

from docklib import Dock

# Load current Dock
dock = Dock()

app_changes = {
    # Apps to be added
    "add": [
        "/Applications/Managed Software Center.app",
    ],
    # Apps to be removed
    "remove": [
        "Calendar",
        "Contacts",
        "Reminders",
        "FaceTime",
    ],
    # Apps (0) to be replaced with other apps (1)
    "replace": [
        ("Safari", "/Applications/Microsoft Edge.app"),
        ("Mail", "/Applications/Microsoft Outlook.app"),
        ("Pages", "/Applications/Microsoft Word.app"),
        ("Numbers", "/Applications/Microsoft Excel.app"),
        ("Keynote", "/Applications/Microsoft PowerPoint.app"),
        ("Messages", "/Applications/Microsoft Teams.app"),
        ("Notes", "/Applications/Microsoft OneNote.app"),
    ],
}

# Apply the adds/removes/replacements
for app in app_changes["add"]:
    item = dock.makeDockAppEntry(app)
    dock.items["persistent-apps"].append(item)
for app in app_changes["remove"]:
    dock.removeDockEntry(app)
for app in app_changes["replace"]:
    dock.replaceDockEntry(app[1], app[0])

# Save changes and relaunch Dock
dock.save()
```

Although you're giving up some control of the items’ order, the main advantage here is that you're only making the changes you _need_. Things you don't care about in the default Dock are left alone rather than wiped out.

To that end: {{% mark %}}Do you want Apple's newly featured apps to be easily discoverable by your users?{{% /mark %}} New major versions of macOS often have new featured applications in the Dock. (Previous examples of such apps have been News, Podcasts, Maps, and TV.) The dynamic example above _will_ allow these featured applications to remain in the Dock untouched.

{{< admonition tip "Dock fixup plist" >}}
The once-per-user adjustment to the Dock that occurs when logging in to a newly installed or upgraded major macOS version is called a "fixup." The plist file that defines the fixup items for the current macOS version can be found here:

    /System/Library/CoreServices/Dock.app/Contents/Resources/com.apple.dockfixup.plist
{{< /admonition >}}

## Skip missing apps

Another question: {{% mark %}}Will all these apps be installed by the time your Dock script runs?{{% /mark %}} Usually, the answer is yes, especially if your provisioning process uses [Munki's bootstrap mode](https://github.com/munki/munki/wiki/Bootstrapping-With-Munki) or a [DEPNotify workflow](https://gitlab.com/Mactroll/DEPNotify#workflow) to trigger your initial software installations prior to first login.

But if the apps don't land on disk for some reason, you may end up with question marks in the Dock for any missing items. These question marks will fix themselves if the referenced app is subsequently installed, and the icon will update when clicked (or when the Dock restarts via a logout, restart, or `killall Dock` command) — but that's not a great user experience.

If there's any doubt as to whether your defined apps will be installed at runtime, you can add some resilience to your script by using `os.path.isdir` to check for the existence of each app and only add the ones that exist on disk, shown in lines 22-23 below.

### Example: Static list of installed apps  <!-- omit in toc -->

```py {linenos=table, hl_lines=["22-23"], linenostart=1}
#!/usr/local/bin/managed_python3

import os
from docklib import Dock

# Load current Dock
dock = Dock()

# Define list of apps, from left to right
desired_apps = [
    "/System/Applications/Launchpad.app",
    "/Applications/Google Chrome.app",
    "/Applications/Microsoft Outlook.app",
    "/Applications/Slack.app",
    "/Applications/Managed Software Center.app",
    "/System/Applications/System Preferences.app",
]

# Set persistent-apps as desired
p_apps = []
for app in desired_apps:
    if os.path.isdir(app):
        p_apps.append(dock.makeDockAppEntry(app))
dock.items["persistent-apps"] = p_apps

# Save changes and relaunch Dock
dock.save()
```

For extra Python flavor, that `for` loop can be compressed into a [list comprehension](https://docs.python.org/3/tutorial/datastructures.html#list-comprehensions), as long as you don't find those unreadable:

```py {linenos=table, hl_lines=["20-22"], linenostart=1}
#!/usr/local/bin/managed_python3

import os
from docklib import Dock

# Load current Dock
dock = Dock()

# Define list of apps, from left to right
desired_apps = [
    "/System/Applications/Launchpad.app",
    "/Applications/Google Chrome.app",
    "/Applications/Microsoft Outlook.app",
    "/Applications/Slack.app",
    "/Applications/Managed Software Center.app",
    "/System/Applications/System Preferences.app",
]

# Set persistent-apps as desired
dock.items["persistent-apps"] = [
    dock.makeDockAppEntry(x) for x in desired_apps if os.path.isdir(x)
]

# Save changes and relaunch Dock
dock.save()
```

## Avoid monolingual assumptions

Another very important consideration: {{% mark %}}What language will your users be using when your Dock script runs?{{% /mark %}} During development of a recent Dock script, a colleague made me aware of some English-centric assumptions that I had baked into my script (which were also [present in docklib itself](https://github.com/homebysix/docklib/issues/32)).

Specifically: using a Dock item's "label" to perform find/replace/remove operations will be unreliable if the logged-in user isn't using the language you're writing for. To illustrate, here's the Messages app label in Big Sur when Spanish is selected as the system language:

![Mensajes](../images/resilient-docklib-mensajes.png)

If we take a look at the (truncated) plist data available for the Messages Dock item, we can see that although the label has been localized for the selected language, the filesystem path (represented by `_CFURLString`) remains unchanged:

```xml {linenos=table, hl_lines=["10-11","15-16"], linenostart=1}
<dict>
    <key>GUID</key>
    <integer>1092548092</integer>
    <key>tile-data</key>
    <dict>
        <key>bundle-identifier</key>
        <string>com.apple.MobileSMS</string>
        <key>file-data</key>
        <dict>
            <key>_CFURLString</key>
            <string>file:///System/Applications/Messages.app/</string>
            <key>_CFURLStringType</key>
            <integer>15</integer>
        </dict>
        <key>file-label</key>
        <string>Mensajes</string>
    </dict>
    <key>tile-type</key>
    <string>file-tile</string>
</dict>
```

Consequently, I've made some changes to docklib as of [version 1.3.0](https://github.com/homebysix/docklib/releases/tag/v1.3.0) that help address this. The bottom line: functions that previously depended solely on item labels now (by default) take `_CFURLString` into account first.

If you're delving into the `persistent-apps` items yourself rather than using the docklib functions, just remember not to rely on `item["tile-data"]["file-label"]`.

## Define the scope and conditions

Next: {{% mark %}}What machines/users are in-scope for your Dock script?{{% /mark %}} Your endpoint management or MDM tool can likely be used to target a specific subset of Macs, or Macs at a specific point in their lifecycle. Additionally, if you have a central directory system you can also leverage identity data for scoping.

Regardless of how the scope is determined by endpoint management tools, I like to build some basic scope limitations into the script itself. Even though this creates some redundancy, the habit has proven useful in limiting damage from upstream misconfigurations.

### Example: Limit to certain hostname patterns  <!-- omit in toc -->

If your endpoint management tool applies a predictable hostname scheme to provisioned Macs, it's possible to leverage this information to elect in or out of the Dock configuration at script runtime. Here's one way to do that:

```py {linenos=table, hl_lines=["7-11"], linenostart=1}
#!/usr/local/bin/managed_python3

import socket
import sys
from docklib import Dock

# Modify Dock only on hosts that start with M_
hostname = socket.gethostname()
if not hostname.startswith("M_"):
    print("Hostname (%s) is out of scope for Dock config. Exiting." % hostname)
    sys.exit(0)

# Load current Dock
dock = Dock()

### CODE THAT ACTUALLY MODIFIES DOCK WOULD GO HERE ###

# Save changes and relaunch Dock
dock.save()
```

### Example: Conditional Dock items based on hostname  <!-- omit in toc -->

{{% mark %}}Do you need to apply different Docks to specific sets of Macs?{{% /mark %}} Creating two or more slightly-different Dock scripts and scoping them "just so" on your endpoint management tool can be tedious and difficult to audit. An alternative (or complementary) solution would be to build the conditions into the script's logic.

Building on the example above, here's a script that would apply Docks based on computer hostname:

```py {linenos=table, hl_lines=["10-38"], linenostart=1}
#!/usr/local/bin/managed_python3

import os
import socket
from docklib import Dock

# Load current Dock
dock = Dock()

# Define list of apps and autohide based on hostname pattern
hostname = socket.gethostname()
if hostname.startswith("mac-build"):
    desired_apps = [
        "/Applications/Xcode.app",
        "/System/Applications/Utilities/Activity Monitor.app",
        "/System/Applications/Utilities/Console.app",
        "/System/Applications/Utilities/Terminal.app",
    ]
elif hostname.startswith("mac-dash"):
    desired_apps = [
        "/Applications/Google Chrome.app",
    ]
    dock.autohide = True
elif hostname.startswith("mac-av"):
    desired_apps = [
        "/System/Applications/QuickTime Player.app",
        "/System/Applications/VLC.app",
    ]
    dock.autohide = True
else:
    desired_apps = [
        "/System/Applications/Launchpad.app",
        "/Applications/Google Chrome.app",
        "/Applications/Microsoft Outlook.app",
        "/Applications/Slack.app",
        "/Applications/Managed Software Center.app",
        "/System/Applications/System Preferences.app",
    ]

# Set persistent-apps as desired
dock.items["persistent-apps"] = [
    dock.makeDockAppEntry(x) for x in desired_apps if os.path.isdir(x)
]

# Save changes and relaunch Dock
dock.save()
```

### Example: Conditional Dock items based on username  <!-- omit in toc -->

{{% mark %}}Do you need to apply different Docks to specific <em>users</em>?{{% /mark %}} Although the practice is falling out of favor for [very good reasons](https://grahamgilbert.com/blog/2020/11/13/a-pragmatic-approach-to-endpoint-security/), many organizations still maintain a local administrator account for IT support technicians. Such an account might benefit from troubleshooting tools like Activity Monitor, Disk Utility, and Terminal in its Dock. Similar customizations may be beneficial for other single-purpose logins like software builders, audio-video, and wall dashboards.

This example applies different Dock configurations depending on the current user:

```py {linenos=table, hl_lines=["10-41"], linenostart=1}
#!/usr/local/bin/managed_python3

import getpass
import os
from docklib import Dock

# Load current Dock
dock = Dock()

# Define list of apps and autohide based on username
username = getpass.getuser()
if username == "itadmin":
    desired_apps = [
        "/System/Applications/Launchpad.app",
        "/Applications/Google Chrome.app",
        "/Applications/Malwarebytes.app",
        "/System/Applications/Utilities/Activity Monitor.app",
        "/System/Applications/Utilities/Console.app",
        "/System/Applications/Utilities/Disk Utility.app",
        "/System/Applications/Utilities/Terminal.app",
    ]
elif username == "dashboard":
    desired_apps = [
        "/Applications/Google Chrome.app",
    ]
    dock.autohide = True
elif username == "av":
    desired_apps = [
        "/System/Applications/QuickTime Player.app",
        "/System/Applications/VLC.app",
    ]
    dock.autohide = True
else:
    desired_apps = [
        "/System/Applications/Launchpad.app",
        "/Applications/Google Chrome.app",
        "/Applications/Microsoft Outlook.app",
        "/Applications/Slack.app",
        "/Applications/Managed Software Center.app",
        "/System/Applications/System Preferences.app",
    ]

# Set persistent-apps as desired
dock.items["persistent-apps"] = [
    dock.makeDockAppEntry(x) for x in desired_apps if os.path.isdir(x)
]

# Save changes and relaunch Dock
dock.save()
```

## Consider user context

Speaking of logged-in users: {{% mark %}}How will I ensure my script executes in the proper context?{{% /mark %}} Scripts that modify the Dock typically need to run in the context of the logged-in user, but most endpoint management systems run scripts in root context by default.

The details of executing Dock scripts in user context are outside the scope of this post, but three possible methods to evaluate include:

- [Outset](https://github.com/chilcote/outset), a tool purpose-built for running scripts at login in user context
- Leverage `sudo`/`launchctl` (see Armin Briegel's post ["Running a Command as Another User"](https://scriptingosx.com/2020/08/running-a-command-as-another-user/))
- Create and deploy your own LaunchAgent that triggers your Dock script

## Enable idempotence

{{% mark %}}Do you need your script to be able to run multiple times without adverse effects?{{% /mark %}} Depending on the method you use to deploy your script, you may need to enable your script to run at _every_ login rather than just the _first_ or _next_ login for each user.

When writing Dock scripts I aim to make them as [idempotent](https://en.wikipedia.org/wiki/Idempotence) as possible, for two main reasons. First, it's simpler to create a script capable of running at every login, versus creating a robust solution for tracking which users have already run the Dock script. Second, I just sleep better at night if I know the script I'm deploying is unlikely to result in any user configurations being overwritten.

In the past, I would simulate idempotence by having the script use the presence of a "flag" file or a preference key to determine whether to proceed with making changes. However, I encountered various problems with this approach and found it cumbersome to test effectively.

Recently I've switched to a more thorough introspection of the current Dock contents to determine whether it's customized or still unmodified from the macOS default, as seen in the example below.

### Example: Only modify uncustomized Docks  <!-- omit in toc -->

This script stores a list of all the apps that have been present in modern macOS default Docks. If the current Dock consists _solely_ of items in that list, the Dock is probably safe to alter in my estimation.

<!-- NOTE: Periodically verify and update the following list of default macOS apps, and
sync the list to this gist: https://gist.github.com/homebysix/077b373264a101d84a3f8cb48e9bca84 -->

```py {linenos=table, hl_lines=["12-19", "29-33", "42-43", "52-55"], linenostart=1}
#!/usr/local/bin/managed_python3

import os
import sys
from urllib.parse import urlparse, unquote
from docklib import Dock


def is_default(dock):
    """Return True if the dock is uncustomized from macOS default, or False otherwise."""

    # List of default Dock items from recent versions of macOS
    # fmt: off
    apple_default_apps = [
        "App Store", "Calendar", "Contacts", "FaceTime", "Finder", "Freeform",
        "iBooks", "iCal", "iPhone Mirroring", "iTunes", "Keynote", "Launchpad",
        "Mail", "Maps", "Messages", "Mission Control", "Music", "News", "Notes",
        "Numbers", "Pages", "Photo Booth", "Photos", "Podcasts", "Reminders",
        "Safari", "Siri", "System Preferences", "TV",
    ]
    # fmt: on

    # Gather a list of default/custom apps for script output
    apps = {"default": [], "custom": []}
    for item in dock.items.get("persistent-apps", []):
        try:
            # Compare the path, not the label, due to localization
            pathurl = item["tile-data"]["file-data"]["_CFURLString"]
            path = urlparse(unquote(pathurl)).path.rstrip("/")
            app_name = os.path.split(path)[-1].replace(".app", "")
            # Add each app into either "custom" or "default" list
            if app_name in apple_default_apps:
                apps["default"].append(app_name)
            else:
                apps["custom"].append(app_name)
        except Exception as err:
            print("ERROR: Exception encountered when processing an item:\n%s" % item)
            print("Raising traceback and leaving Dock unchanged...")
            raise err

    print("Apple default apps: %s" % ", ".join(apps["default"]))
    print("Custom apps: %s" % ", ".join(apps["custom"]))

    # Dock is default if no custom apps were found
    return apps["custom"] == []


def main():
    """Main process."""

    # Load current Dock
    dock = Dock()

    # Bail out now if the Dock appears to be customized by the user
    if not is_default(dock):
        print("Dock appears to be customized already. Exiting.")
        sys.exit(0)

    ### CODE THAT ACTUALLY MODIFIES DOCK WOULD GO HERE ###

    # Save changes and relaunch Dock
    dock.save()


if __name__ == "__main__":
    main()
```

Of course, some holes exist in this logic: What if somebody genuinely only uses Apple's first-party apps for everything, and their customized Dock is very close to the default? What if somebody removes _every_ app from their Dock, preferring it to be empty? These edge cases can be handled in the code if desired, but so far I've chosen not to.

Additionally, the above approach requires a bit of maintenance: administrators to pay attention to new macOS releases and add new default apps to the `apple_default_apps` list as needed.

I'd like to call attention to lines 25-28 of the above code for a moment:

```py {linenos=table, linenostart=25}
# Compare the path, not the label, due to localization
pathurl = item["tile-data"]["file-data"]["_CFURLString"]
path = urlparse(unquote(pathurl)).path.rstrip("/")
app_name = os.path.split(path)[-1].replace(".app", "")
```

I'm taking [my own advice](#avoid-monolingual-assumptions) here by not relying on the item's label as an accurate point for comparison to our "Apple default apps" list. This allows the `is_default` function to work as expected regardless of the user's selected language.

## Prevent race conditions

I've occasionally encountered a [race condition](https://en.wikipedia.org/wiki/Race_condition) with docklib scripts wherein the script executes before the Dock itself has launched. On a user's first login, this results in a situation where docklib tries to read a preference that doesn't yet exist, causing the script to fail.

The way dockutil [worked around this issue](https://github.com/kcrawford/dockutil/commit/09f132e7cfed4d26eb0808f877df089e59ec978a) (when it was written in Python) was by waiting for the `mod-count` of the dock to be greater than 1. Docklib takes a more hands-off approach, choosing to leave this particular workaround up to admins’ discretion instead.

I prefer to build in a loop that waits for the Dock process to run before continuing.

```py {linenos=table, hl_lines=["9-27", "33-34"], linenostart=1}
#!/usr/local/bin/managed_python3

from time import sleep
import subprocess
import sys
from docklib import Dock


def wait_for_dock(max_time=60):
    """Wait for Dock to launch. Bail out if we reach max_time seconds."""

    count = 0
    check_cmd = ["/usr/bin/pgrep", "-qx", "Dock"]

    # Check every 1 second for the Dock process
    while subprocess.run(check_cmd, check=False).returncode != 0:
        if count >= max_time:
            # We reached our max_time
            print("Dock did not start within %s seconds. Exiting." % max_time)
            sys.exit(1)
        elif count % 5 == 0:
            # Provide status output every 5 seconds
            print("Waiting up to %d seconds for Dock to start..." % max_time - count)

        # Increment count and wait one second before looping
        count += 1
        sleep(1)


def main():
    """Main process."""

    # Wait maximum 60 seconds for Dock to start
    wait_for_dock(60)

    # Load current Dock
    dock = Dock()

    ### CODE THAT ACTUALLY MODIFIES DOCK WOULD GO HERE ###

    # Save changes and relaunch Dock
    dock.save()


if __name__ == "__main__":
    main()
```

Others have come up with creative solutions to this issue as well:

- Waiting for both the Finder app and the Dock app, as shown [here](https://community.jamf.com/t5/jamf-pro/enrollment-complete-policies-not-running-depnotify/m-p/232674/highlight/true#M220705)
- Using AppKit's `NSRunningApplication` to perform the check, as shown [here](https://gist.github.com/chrisgrande/d3be9102e805acb8380a8b40c7c8421d#file-dock_lab_setup-py-L16-L24)
- Using `killall -s`, a "no-signal" kill command, to perform the check, as shown [here](https://macadmins.slack.com/archives/C0HLW2QAH/p1624649403112300?thread_ts=1624644804.074900&cid=C0HLW2QAH)

No matter which method you use, the end result should be that the Dock modifications are more reliable, especially on slower Macs where the Dock may take a few seconds to launch.

## Make backups

{{% mark %}}Is there any chance you'd need to revert the changes your script makes to users’ Docks?{{% /mark %}} To be safe, you can implement a function that performs a backup of your Dock plist prior to saving a new one, like so:

```py {linenos=table, hl_lines=["12-22"], linenostart=1}
#!/usr/local/bin/managed_python3

import os
import shutil
from datetime import datetime
from docklib import Dock


# Load current Dock
dock = Dock()

# Make a backup of current Dock plist
dock_plist = os.path.expanduser("~/Library/Preferences/com.apple.dock.plist")
backup_dir = os.path.expanduser("~/Library/PretendCo/backup/")
if os.path.isfile(dock_plist):
    if not os.path.isdir(backup_dir):
        os.makedirs(backup_dir)
    datestamp = datetime.strftime(datetime.now(), "%Y-%m-%d %H-%M-%S")
    shutil.copy(
        dock_plist,
        os.path.join(backup_dir, "com.apple.dock (%s).plist" % datestamp),
    )

### CODE THAT ACTUALLY MODIFIES DOCK WOULD GO HERE ###

# Save changes and relaunch Dock
dock.save()
```

If a restore is needed, the appropriate plist could be moved from `~/Library/PretendCo/backup/com.apple.dock (<datestamp>).plist` to `~/Library/Preferences/com.apple.dock.plist`.

## Simplify troubleshooting

In all the examples above, I've tried to consistently leave [comments](https://www.python.org/dev/peps/pep-0008/#comments) that explain what each section of code is intended to do. You'd be well-served to do the same — your future self will thank you, not to mention your colleagues who may need to tweak your script down the road.

For more complex scripts that include functions for performing specific tasks, include a [docstring](https://www.python.org/dev/peps/pep-0257/) to explain what each function does.

In addition to comments and docstrings, log output can be tremendously useful. One option is to create a function for generating log messages with timestamps, as shown below. Call this function whenever the script performs a notable action.

```py {linenos=table, hl_lines=["7-10", 16, 19, 22], linenostart=1}
#!/usr/local/bin/managed_python3

from datetime import datetime, timezone
from docklib import Dock


def output(message):
    """Write a message to standard out, prefixed with a datestamp."""
    datestamp = datetime.strftime(datetime.now(timezone.utc), "%Y-%m-%d %H:%M:%S %z")
    print("%s [dock-script.py]: %s" % (datestamp, message))


def main():
    """Main process."""

    output("Loading current Dock...")
    dock = Dock()

    output("Making changes to Dock...")
    ### CODE THAT ACTUALLY MODIFIES DOCK WOULD GO HERE ###

    output("Saving and relaunching Dock...")
    dock.save()


if __name__ == "__main__":
    main()
```

Note that Python contains a [`logging` module](https://docs.python.org/3/library/logging.html) that you can leverage if you prefer. This function allows you to set the [level](https://docs.python.org/3/library/logging.html#logging-levels) of a message, format output consistently, and other conveniences.

```py {linenos=table, hl_lines=["7-10", 12, 16, 19, 22], linenostart=1}
#!/usr/local/bin/managed_python3

import logging
import sys
from docklib import Dock

# Set up logging config
logging.basicConfig(
    level=logging.INFO, format="%(asctime)s [%(filename)s] %(levelname)s: %(message)s"
)

logging.info("Loading current Dock...")
dock = Dock()

if not dock:
    logging.error("Unable to load Dock...")
    sys.exit(1)

logging.info("Making changes to Dock...")
### CODE THAT ACTUALLY MODIFIES DOCK WOULD GO HERE ###

logging.info("Saving and relaunching Dock...")
dock.save()
```

If you use a custom LaunchAgent to trigger your Dock script, you can define the agent's `StandardErrorPath` and `StandardOutPath` to determine where on disk this output is stored. (Be sure the user running the script has permission to write to the output location.) If you're using Outset, the output for user-context scripts is stored in `~/Library/Logs/outset.log`.

## Tying it all together

At last, here's a fully-featured example docklib script that incorporates many of the resilience tips demonstrated above. Feel free to use this script as a launching-off point for your own Dock customization adventures.

Once your script is working as expected on your test Mac, the next step would be to test and deploy it (along with any required frameworks like Outset and Python 3) on your endpoint Macs. I've written [another post](../docklib-outset/) that describes that process.

{{% gist homebysix 077b373264a101d84a3f8cb48e9bca84 %}}
