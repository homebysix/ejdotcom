---
title: Managing login mechanisms in the macOS authorization database
date: 2023-08-05T06:00:00-07:00
description: Advice for Mac systems engineers who are deploying authorization plugins to their managed Macs and need to preserve the desired state of `system.login.console` mechanisms in the macOS authorization database.
slug: macos-authdb-mechs
tags:
    - macadmin
    - authdb
    - shell
    - escrow buddy
---

The process of creating and publishing a new open-source macOS authorization [plugin](https://github.com/macadmins/escrow-buddy/) has taught me a few things about handling the list of login mechanisms in the authorization database. It also helped me realize that existing methods of maintaining the database often rely upon Python — a dependency I preferred to avoid.

In this post, I'll go over the basics of managing the login mechanism list using shell scripts, which may be of interest to organizations who deploy [Escrow Buddy](https://github.com/macadmins/escrow-buddy/), [Crypt](https://github.com/grahamgilbert/crypt/), or [XCreds](https://twocanoes.com/products/mac/xcreds/). I'll also share a script that includes flexible shell functions for those who manage authorization plugins but don't want or need to deploy a Python runtime.

## Authorization plugins

macOS provides [numerous ways](https://theevilbit.github.io/categories/beyond/) for developers and IT administrators to run processes in the background. The most common ones — including LaunchDaemons and LaunchAgents — are widely understood, used, and abused in the Mac admin and security community. But there are situations where a daemon or agent isn't the ideal tool for the job, and one of these situations is automation and decision-making around user login.

Enter [macOS authorization plugins](https://developer.apple.com/documentation/security/authorization_plug-ins). Apple's documentation describes their use:

> A typical use for authorization plug-ins is to implement policies that are not included in the standard authorization configuration.

In the Mac admin world, such policies could include federating local accounts to cloud identity providers, providing additional account setup automation during the provisioning process, or handling situations that require user credentials like keychain syncing and FileVault key generation.

## Authorization database

Authorization plugins can have multiple "mechanisms" that can serve specific purposes or run with specific privileges. In order to register these mechanisms with the macOS authorization services API, they need to be listed in the `system.login.console` section of the authorization database, within the `mechanisms` array.

The command `security authorizationdb read system.login.console` will display the current mechanisms in a plist. Unlike many on-disk plists Mac admins are used to dealing with, this plist exists in memory. (Although files do exist at `/var/db/auth.db*` that appear to cache the parsed contents of the database, directly rewriting these files is not recommended by Apple.)

<!-- NOTE: Periodically verify and update the following command output. -->

As of macOS Ventura, the command above produces the following output:

```xml
% security authorizationdb read system.login.console
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>class</key>
    <string>evaluate-mechanisms</string>
    <key>comment</key>
    <string>Login mechanism based rule.  Not for general use, yet.</string>
    <key>created</key>
    <real>470991139.20000899</real>
    <key>mechanisms</key>
    <array>
        <string>builtin:prelogin</string>
        <string>builtin:policy-banner</string>
        <string>loginwindow:login</string>
        <string>builtin:login-begin</string>
        <string>builtin:reset-password,privileged</string>
        <string>loginwindow:FDESupport,privileged</string>
        <string>builtin:forward-login,privileged</string>
        <string>builtin:auto-login,privileged</string>
        <string>builtin:authenticate,privileged</string>
        <string>PKINITMechanism:auth,privileged</string>
        <string>builtin:login-success</string>
        <string>loginwindow:success</string>
        <string>HomeDirMechanism:login,privileged</string>
        <string>HomeDirMechanism:status</string>
        <string>MCXMechanism:login</string>
        <string>CryptoTokenKit:login</string>
        <string>PSSOAuthPlugin:login-auth</string>
        <string>loginwindow:done</string>
    </array>
    <key>modified</key>
    <real>692230356.95700204</real>
    <key>shared</key>
    <true/>
    <key>tries</key>
    <integer>10000</integer>
    <key>version</key>
    <integer>10</integer>
</dict>
</plist>
YES (0)
```

The output has two parts: a plist representing the current state of the authorization database (sent to the *stdout* [pipe](https://support.apple.com/lt-lt/guide/terminal/apd1dbe647b-7e11-49dc-aa76-89aa7e53ce36/mac)), and a status message indicating whether the query was successful (to *stderr*). Here's an example of an unsuccessful query generating no *stdout* and an error code in *stderr*:

```
% security authorizationdb read bogus-right-name
NO (-60005)
```

## Resets can happen

{{% mark %}}During some macOS upgrades and updates, the list of login mechanisms in the authorization database gets reset to its default state.{{% /mark %}} This is understandable, as the introduction of new macOS features sometimes requires new entries in the database, and Apple probably aims to avoid the risk of incompatible third-party mechanisms breaking the login process.

{{< admonition tip "Default authorization database" >}}
The fresh authorization database is synthesized from <em>/System/Library/Security/authorization.plist</em>. Referring to this file on a given version of macOS can help you understand which mechanisms are standard and in which order they're found.
{{< /admonition >}}

If you deploy an authorization plugin to your managed Macs, the potential for periodic reset puts the onus on you and your team to be aware of when the plugins you deploy might have been removed from the authorization database and — after testing and ensuring compatibility with new macOS versions — re-adding your plugin mechanism to the authorization database to restore functionality.

## Auditing desired state

In order to check whether your custom mechanism is present in the authorization database, a simple `grep` can provide the necessary logic:

```
% security authorizationdb read system.login.console | grep "<string>YourPlugin:Mechanism</string>"
    <string>YourPlugin:Mechanism</string>
YES (0)
```

If you're planning on including this logic in a script, you may not want any output. To suppress output, we can add `-q` to the `grep` command, and redirect *stderr* to /dev/null.

```
% security authorizationdb read system.login.console 2>/dev/null | grep -q "<string>YourPlugin:Mechanism</string>"
```

This will produce a predictable exit code: zero if the desired mechanism is present, non-zero otherwise. An example of using this in a script-based collection mechanism (like a Jamf extension attribute) would be:

{{< gist homebysix 6aa2326a741d330a4b0a3c767a1321e1 escrow_buddy_authdb_status.sh >}}

However, sometimes knowing whether the mechanism is *in* the database isn't sufficient. Because the order of the mechanisms in the array represent the order they're executed during login, you may want to capture where in that array your mechanism is.

This example extension attribute returns the index (beginning at zero) of the specified mechanism, or `-1` if the mechanism isn't listed:

{{< gist homebysix 6aa2326a741d330a4b0a3c767a1321e1 escrow_buddy_authdb_index.sh >}}

## Modifying the mechanism list

If your desired entry is missing from the list of login mechanisms in the authorization database, and you've tested to ensure compatibility with the current macOS version, you can provide an updated plist as input to the `security` command to add your entry.

### Making a backup

Before making any changes, it's not a bad idea to save a backup of the plist in case restoration is required. We can do that by saving the plist to disk and making a copy:

```sh
security authorizationdb read system.login.console 2>/dev/null > /tmp/system.login.console.plist
cp /tmp/system.login.console.plist /tmp/system.login.console.plist.backup
```

Then we can edit the _system.login.console.plist_ file as needed.

### Validating input

Because errors in the authorization database can potentially prevent login from succeeding, extra diligence in catching errors is warranted. Login mechanism identifiers follow a predictable format that we can validate:

![PluginName:Mechanism,privileged](../images/authdb-mech-format.png)

We can use `grep` to ensure any provided mechanism names match a regular expression:

```sh
MECHANISM="YourPlugin:Mechanism"
if ! grep -qE '^[^,:]+:[^,:]+(,privileged)?$' <<< "$MECHANISM"; then
    echo "ERROR: Specified right is not formatted correctly: $MECHANISM"
    exit 1
fi
```

### Relative positioning

When adding mechanisms to the array, we'll need to specify an index; the new mechanism will be inserted into the array at that index, incrementing subsequent items' indices by 1. Therefore, it may be useful to reference an existing item in the array that we can insert relative to. In the example below, we're calculating the index of the built-in `loginwindow:done` mechanism.

```sh
INDEX=$(/usr/libexec/PlistBuddy -c "Print :mechanisms:" /tmp/system.login.console.plist 2>/dev/null | grep -n "loginwindow:done" | awk -F ":" '{print $1}')
INDEX=$((INDEX-2))
```

### Writing changes

Finally, we can use PlistBuddy's `Add` verb to modify the plist on disk, which we can feed as input to the `security authorizationdb write system.login.console` command to apply the changes to the authorization database:

```sh
/usr/libexec/PlistBuddy -c "Add :mechanisms:$INDEX string 'MyPlugin:Action'" /tmp/system.login.console.plist
security authorizationdb write system.login.console < /tmp/system.login.console.plist
```

### Removing mechanisms

If we need to remove a mechanism from the list, we can use a similar process to do so. First we'll find its index. Then we'll remove it using PlistBuddy's `Delete` verb. Then we'll write the plist back to the database.

```sh
INDEX=$(/usr/libexec/PlistBuddy -c "Print :mechanisms:" /tmp/system.login.console.plist 2>/dev/null | grep -n "MyPlugin:Action" | awk -F ":" '{print $1}')
INDEX=$((INDEX-2))
/usr/libexec/PlistBuddy -c "Delete :mechanisms:$INDEX" /tmp/system.login.console.plist
security authorizationdb write system.login.console 2>/dev/null < /tmp/system.login.console.plist
```

## Tying it all together

In order to simplify the process of managing `system.login.console` mechanism entries for IT administrators, I've produced a shell script with various functions for dealing with the authorization database. This script incorporates the concepts covered above, and adds informative output, error handling, and sensible use of variables to make customization and implementation easier.

You can find this script [on GitHub](https://gist.github.com/homebysix/6aa2326a741d330a4b0a3c767a1321e1#file-modifyauthdbloginmechs-sh) and embedded below. To use it, uncomment and customize the examples at the bottom of the script to suit your needs.

{{< gist homebysix 6aa2326a741d330a4b0a3c767a1321e1 ModifyAuthDBLoginMechs.sh >}}

## Further reading

If you're interested in learning more about managing the `system.login.console` right of the macOS authorization database, check out the articles and code examples below:

- Csaba Fitzl: [Beyond the good ol' LaunchAgents - 28 - Authorization Plugins](https://theevilbit.github.io/beyond/beyond_0028/)
- DssW: [system.login.console - Overview of the macOS Authorization Right](https://www.dssw.co.uk/reference/authorization-rights/system-login-console/) (includes default mechanisms for macOS 10.6 through 10.14)
- Charles Edge: [authorizationdb Defaults in macOS 10.14](https://krypted.com/utilities/authorizationdb-defaults-macos-10-14/#:~:text=system.login.console)
- Erik Berglund: [Scripts/random/modifyAuthorizationDB/modifyAuthorizationDB](https://github.com/erikberglund/Scripts/blob/master/random/modifyAuthorizationDB/modifyAuthorizationDB)
- Jamf: [Modifying the loginwindow Application](https://learn.jamf.com/bundle/jamf-connect-documentation-current/page/Editing_the_macOS_loginwindow_Application.html)
- JumpCloud: [support/scripts/macos/remove_mac_agent.sh](https://github.com/TheJumpCloud/support/blob/fc97cb18df196324b80156bf2a825fee9ebb36a4/scripts/macos/remove_mac_agent.sh#L29-L31) (clever one-line removal method using `sed`)
- Airbnb: [macadmins/puppet-authpluginmech: A method of managing mechanisms for authorization plugins](https://github.com/macadmins/puppet-authpluginmech)
- Airbnb: [puppet-crypt/manifests/config.pp](https://github.com/airbnb/puppet-crypt/blob/master/manifests/config.pp#L66-L79)

And if you're interested in the other rights managed by the authorization database, see:

- Rich Trouton: [Managing the Authorization Database in OS X Mavericks](https://derflounder.wordpress.com/2014/02/16/managing-the-authorization-database-in-os-x-mavericks/)
- Armin Briegel: [Demystifying `root` on macOS, Part 4 —The Authorization Database](https://scriptingosx.com/2018/05/demystifying-root-on-macos-part-4-the-authorization-database/)
- Nathaniel Strauss: [Authorization Rights Management for Standard User Access](https://nwstrauss.com/posts/2021-01-28-managing-authorizationdb/)
- Apple: [Introduction to Authorization Services Programming Guide](https://developer.apple.com/library/archive/documentation/Security/Conceptual/authorization_concepts/01introduction/introduction.html) (archived, but helpful context)
