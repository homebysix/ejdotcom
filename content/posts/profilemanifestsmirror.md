---
title: "ProfileCreator Manifests Now Available for Jamf"
date: 2021-12-16T19:00:00-08:00
description: Introducing a mirror of the ProfileManifests repository in JSON schema format, which provides Jamf administrators access to an extensive collection of manifests that can be used with the custom manifest feature of Jamf.
slug: profilemanifestsmirror
tags:
    - mdm
    - jamf
    - macadmin
    - python
---

Many of my favorite [projects](/projects/) have come to fruition by following a very specific feeling: that something useful _could_ be automated but _hasn't_ been yet. It's a powerful motivator to imagine how much time might be saved by making manual processes automatic, and then to realize that I might have the tools available to engineer that automation.

I got that feeling again while watching [Leonardo Cacioppo's presentation about custom app management manifests](https://www.youtube.com/watch?v=3ZdFzWBTkjg) from the Jamf Nation User Conference. I recommend watching the video to understand the details, but the basic idea is that providing a custom JSON manifest allows Jamf administrators to [configure profiles for app settings](https://developer.jamf.com/developer-guide/docs/application-and-custom-settings) in a much more accessible and error-proof way than hand-editing and uploading raw plist files. A few people in the Mac admin community have started [sharing their JSON manifests on GitHub](https://github.com/Jamf-Custom-Profile-Schemas) for others to use.

As soon I understood the details, I immediately drew parallels to the [ProfileManifests](https://github.com/ProfileCreator/ProfileManifests) repository, which both [ProfileCreator](https://github.com/ProfileCreator/ProfileCreator) and [iMazing Profile Editor](https://imazing.com/profile-editor) use as the basis for their profile definitions. The files in ProfileManifests use Apple's [preference manifest format](https://github.com/ProfileCreator/ProfileManifests/wiki/Manifest-Format). Since both the preference manifests and Jamf's JSON manifests serve the same goal — describing the format of plist files that manage app settings — I hypothesized that it should be possible to convert one to the other.

After diving deep into the documentation available on each format, reviewing dozens of example files, and some Python trial and error, I created a working converter from the preference manifest format to Jamf's JSON schema manifest format. I connected the script to GitHub Actions and created a repository to host the resulting files.

## Introducing the ProfileManifestsMirror repository

{{% mark %}}The new [ProfileManifestsMirror](https://github.com/Jamf-Custom-Profile-Schemas/ProfileManifestsMirror) repository contains Jamf-ready versions of all the manifests that power ProfileCreator and iMazing Profile Editor.{{% /mark %}} Pasting or uploading these manifests to Jamf when creating a new MDM configuration profile reveals an intuitive interface for adjusting the settings in your profile, along with links to helpful documentation where available.

My primary goal for this repository is to give administrators a head start when creating new profiles for apps and domains already included in the extensive ProfileManifests collection.

A secondary goal is to spread awareness of the upstream [ProfileManifests](https://github.com/ProfileCreator/ProfileManifests) repository, so the collection can continue growing to include more apps and stay up to date with changes to existing settings. Focusing contribution energy on this upstream repository benefits everybody in the Mac admin community, including but not limited to those using Jamf.

Finally, I enjoyed having an excuse to build my first useful workflow in GitHub Actions. I found it very approachable, and now I'm looking for ways to use Actions in my other projects.

## Usage

If you'd like to put the ProfileManifestsMirror repository to work for you, here's how:

1. Identify the domain of an app or setting you'd like to manage with an MDM configuration profile. (In this example, I'll use Slack's `SlackNoAutoUpdates` setting, within the `com.tinyspeck.slackmacgap` domain.)

1. Navigate to the [ProfileManifestsMirror repository on GitHub](https://github.com/Jamf-Custom-Profile-Schemas/ProfileManifestsMirror).

1. Type `T` and then a portion of the app's domain. Click on the desired manifest, or use the up/down arrows and type **Return** to select.
    ![Use GitHub file finder to locate a manifest](../images/profilemanifestsmirror-01.png)

1. Click the **Copy Raw Contents** button.
    ![Copy raw contents](../images/profilemanifestsmirror-02.png)

1. Log in to your Jamf web portal and click **Configuration Profiles**, then **New**.

1. In the **General** section, give the profile a name and description.

1. In the **Scope** section, target the profile to your desired Macs (use a test device, initially).

1. Click the **Application & Custom Settings** profile payload, then **External Applications**, then the **Add** button.

1. From the **Source** drop-down menu, choose **Custom Schema**.

1. In the **Preference Domain** field, enter the domain of the app. (For example, `com.tinyspeck.slackmacgap`.)
    ![Set preference domain and add manifest in Jamf](../images/profilemanifestsmirror-03.png)

1. Click the **Add Schema** button. In the **Custom Schema** text field, paste the contents of the manifest you copied from GitHub. Click **Save**.
    ![Add custom manifest](../images/profilemanifestsmirror-04.png)

1. You should now see appropriate fields and menus for each of the settings in the manifest. Adjust the settings as you desire. Click the **More information** link for any setting, if available, to see detailed documentation. You can also click the **Plist preview** tab to see the text of the resulting plist.
    ![Profile settings with custom manifest](../images/profilemanifestsmirror-05.png)

1. Click **Save** when you're satisfied with your profile. Within a few moments, the settings should apply to any online Macs within your configured scope.

As you can see, the visual menus and fields provided by custom profile manifests are more accessible to those who prefer UI elements to editing text plists, while still producing the same output as the existing text-based upload workflow.

## Contributing to manifests

As you use these manifests, you may encounter specific settings or entire domains that are absent from the ProfileManifests repository. When this happens to you, I would strongly encourage you to submit a [pull request](https://github.com/ProfileCreator/ProfileManifests/pulls) to the upstream ProfileManifests repository that adds the missing pieces for the benefit of everyone. ([This pull request](https://github.com/ProfileCreator/ProfileManifests/pull/335/files) that adds a manifest for Handbrake settings is a good example.) After pull requests are reviewed and merged, the ProfileManifestsMirror repository will be updated automatically to reflect the changes.

If you don't have the time or interest for submitting pull requests, the next best thing would be to [check the ProfileManifests repository issues](https://github.com/ProfileCreator/ProfileManifests/issues) to see whether anyone else has already submitted a request for your desired app or setting. If not, submit a new issue, and a generous volunteer may grant your request.

## Contributing to the conversion script

Since the ProfileManifestsMirror repository (and its underlying build script) is new to the scene, it certainly has unfinished features and bugs. If you're Python-savvy, I would welcome pull requests to help make the [conversion script](https://github.com/Jamf-Custom-Profile-Schemas/ProfileManifestsMirror/blob/main/build.py) and [build workflow](https://github.com/Jamf-Custom-Profile-Schemas/ProfileManifestsMirror/blob/main/.github/workflows/build.yml) even better! I've left some `TODO` items in the code, for those looking for a good starting point.

## Spread the word

If you know Jamf admins who would benefit from the manifests contained in the new ProfileManifestsMirror repository, please share a link with them. As more people get involved, more manifests will be contributed that can be used by Jamf admins, ProfileCreator users, and iMazing Profile Editor users alike. As they say, a rising tide lifts all boats.
