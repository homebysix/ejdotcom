---
title: "Removing Jamf Remote policies using an API script"
date: 2023-10-01T10:00:00-07:00
description: An example script that uses the Jamf Pro SDK Python module to remove policies left over from usage of the deprecated Jamf Pro app.
slug: remove-remote-policies
tags:
    - macadmin
    - python
    - jamf
---

The Jamf Remote app was [removed](https://learn.jamf.com/bundle/jamf-pro-release-notes-10.40.0/page/Deprecations_and_Removals.html) as of Jamf Pro version 10.40, but organizations that used the app may still have policies leftover from its use. These policies aren’t visible in the Jamf Pro web interface, but they do appear when making API requests. If you modify or review policies in bulk using the API (for example, using [this script](https://derflounder.wordpress.com/2022/10/14/using-the-jamf-pro-api-to-report-on-self-service-policies/) provided by Rich Trouton or the [Prune](https://github.com/BIG-RAT/Prune) cleanup tool), the presence of these leftover Jamf Remote policies can introduce unnecessary delays and complexity.

Bulk removal of Jamf Remote scripts is possible by searching for policies via the API that match a specific naming convention. I’ve included a script below that uses the new [Jamf Pro SDK](https://github.com/macadmins/jamf-pro-sdk-python) Python module to perform this bulk removal.

{{< gist homebysix ee4a7d615f755ee6eaa68043cdb1ddff >}}

You’ll need to substitute the URL of your Jamf instance in line 16 of the script. When running the script you’ll be prompted for your Jamf Pro credentials. There are [other authentication options](https://macadmins.github.io/jamf-pro-sdk-python/reference/credentials.html) available in the SDK — for example, retrieving the credentials from your macOS keychain or hard-coding the credentials into your copy of the script.

When the script runs, you’ll see confirmation of how many Jamf Remote policies were removed, as well as the name and ID of each policy.
