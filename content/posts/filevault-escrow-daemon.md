---
title: Escrowing FileVault keys to MDM quickly after generation
date: 2023-09-23T13:00:00-07:00
description: A daemon that MDM administrators can use to ensure newly-generated FileVault recovery keys are escrowed to MDM as soon after generation as possible.
slug: filevault-escrow-daemon
tags:
    - macadmin
    - filevault
    - escrow buddy
---

In years past when institutional Macs were primarily managed by a combination of agents, daemons, and scheduled command-line tool invocations, administrators could predict and control the timing of management tasks with much more specificity. The order of related events could be scripted and dependencies could be accounted for reliably and smoothly.

Now that the Mac admin world is moving towards asynchronous MDM and DDM commands for a growing number of management capabilities, it's not as straightforward as it used to be to create workflows that depend upon the successful execution of sequential tasks, especially when mixing tasks delivered over APNS and tasks that use local agents or executables.

A basic example of this difficulty is the gap between the generation of a new FileVault key (using the local `fdesetup` binary built into macOS) and its subsequent escrow to your MDM (during the Mac's response to the `SecurityInfo` MDM command). Many MDMs align the timing of `SecurityInfo` commands to the execution of an inventory collection process, which may happen hours or even days after the FileVault key is escrowed. Even a delay of mere minutes may prove problematic if the affected Mac is not actively used; closing its lid immediately after enabling FileVault could be enough to prevent the new key from being escrowed successfully.

Some administrators have tried to solve this inconvenience by simply running inventory collection more often — at every startup or login, for example. However, more frequent inventory collection comes at the cost of computing and network resources, and the majority of startups and logins are not going to produce actionable changes in device inventory.

## FileVaultPRKWatcher daemon

{{% mark %}}My proposal is to trigger escrow in a more targeted way: by creating a daemon that runs an inventory update when changes to _/var/db/FileVaultPRK.dat_ are detected.{{% /mark %}} This way we can ensure that the inventory update happens as soon after a significant inventory-related event as possible, without producing extra inventory collection runs at other times.

A LaunchDaemon that implements this proposal could be stored at _/Library/LaunchDaemons/com.elliotjordan.FileVaultPRKWatcher.plist_ and contain the following:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Comment</key>
    <string>Details: https://www.elliotjordan.com/posts/{{% param slug %}}/</string>
    <key>Label</key>
    <string>com.elliotjordan.FileVaultPRKWatcher</string>
    <key>ProgramArguments</key>
    <array>
        <string>/usr/local/bin/jamf</string>
        <string>recon</string>
    </array>
    <key>WatchPaths</key>
    <array>
        <string>/var/db/FileVaultPRK.dat</string>
    </array>
</dict>
</plist>
```

Let's break down the components of the LaunchDaemon:

- **`Label`**: A reverse-domain identifier that should match the filename of the daemon. You can customize this to reflect your organization's domain if you prefer.

- **`ProgramArguments`**: An array of strings that represents the shell command that triggers the `SecurityInfo` command for your MDM. I've used `jamf recon`, but you should customize this to fit your MDM. For example, Kandji admins would use `kandji update-mdm`. Be sure each "word" in the command is represented by a string, and use the full path to the binary when possible to avoid relaying on the shell `PATH` environment variable.

- **`WatchPaths`**: This array of strings represents the file path(s) we wish to monitor for changes. In this case, we're specifying the path to the encrypted CMS envelope where macOS stores the new FileVault personal recovery key (PRK).

## Demonstration

To prove this concept, we can use the macOS logs to calculate the time between FileVault key generation and escrow using three different configurations. The command used to retrieve the below logs is:

```sh
log show --style compact \
    --predicate 'process == "mdmclient"
        AND (message CONTAINS "PUT) [Acknowledged(SecurityInfo"
        OR message CONTAINS "Saved PRK escrow file")'
```

### Daily inventory

Here are logs captured on a Jamf-managed Mac configured to update its inventory once per day, which is a typical frequency for Jamf setups:

```txt
Timestamp               Ty Process[PID:TID]
2023-09-06 16:05:19.137 Df mdmclient[276:5b0] [com.apple.ManagedClient:FVEscrow] [0:MDMDaemon:FVEscrow:<0x5b0>] Saved PRK escrow file: YES  Length: 460
2023-09-07 09:50:20.810 Df mdmclient[2873:6149] [com.apple.ManagedClient:HTTPUtil] [0:MDMDaemon:HTTPUtil:<0x6149>] >>>>> Sending HTTP request (PUT) [Acknowledged(SecurityInfo):beebd0b3-7d15-4e06-8894-ba59b7163992] >>>>>
```

In this example, the time between generation and escrow was almost **18 hours**. This could be anything up to 24 hours, but it's likely that this method will produce a delay of hours, not minutes.

### Login inventory

Here are logs captured while configured to run additional inventory updates at login, after using [Escrow Buddy](https://github.com/macadmins/escrow-buddy) to generate a new FileVault recovery key at login:

```txt
Timestamp               Ty Process[PID:TID]
2023-09-23 10:18:50.095 Df mdmclient[22825:83fdf] [com.apple.ManagedClient:FVEscrow] [0:MDMDaemon:FVEscrow:<0x83fdf>] Saved PRK escrow file: YES  Length: 460
2023-09-23 10:19:17.267 Df mdmclient[22825:8502e] [com.apple.ManagedClient:HTTPUtil] [0:MDMDaemon:HTTPUtil:<0x8502e>] >>>>> Sending HTTP request (PUT) [Acknowledged(SecurityInfo):2d7b0d7f-5fcc-4c3a-be58-80ab727cded2] >>>>>
```

Inventory at login produces a much quicker turnaround (less than **1 minute** in this example), but with two downsides. First, inventory collection consumes computing and network resources at *every* login, even when device inventory information has not changed in a substantial way. Second, FileVault keys may be generated at times other than login — during the macOS Setup Assistant, by enabling FileVault in System Settings, or by running `sudo fdesetup changerecovery -personal` manually. Configuring inventory collection at login won't help speed the process in those situations.

### FileVaultPRKWatcher daemon inventory

Finally, here are logs captured on the same Mac running the LaunchDaemon shown above:

```txt
Timestamp               Ty Process[PID:TID]
2023-09-23 10:54:42.597 Df mdmclient[1551:276b] [com.apple.ManagedClient:FVEscrow] [0:MDMDaemon:FVEscrow:<0x276b>] Saved PRK escrow file: YES  Length: 460
2023-09-23 10:55:03.314 Df mdmclient[1551:2cb1] [com.apple.ManagedClient:HTTPUtil] [0:MDMDaemon:HTTPUtil:<0x2cb1>] >>>>> Sending HTTP request (PUT) [Acknowledged(SecurityInfo):30c1cae4-972a-4293-a60b-7cdca34833fc] >>>>>
```

In this example, the time between generation and escrow was **21 seconds**. (Your results may vary, but should be in that ballpark.) Not only does the LaunchDaemon above produce the quickest time to escrow, it also handles escrow after _any_ FileVault key generation, not just those that Escrow Buddy trigger at login.

This concept isn't foolproof — the escrow of keys generated while the Mac is not connected to a network may be delayed until a subsequent `SecurityInfo` response occurs. And this method would not work for escrow methods that don't use a [FDERecoveryKeyEscrow](https://developer.apple.com/documentation/devicemanagement/fderecoverykeyescrow) payload, such as Jamf's FileVault enablement policy or via Crypt.

But for most administrators who escrow FileVault keys to MDM, I hope this method proves valuable to ensure timely retrieval of new FileVault recovery keys.
