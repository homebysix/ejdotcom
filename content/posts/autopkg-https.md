---
title: "Switch AutoPkg recipes to HTTPS"
date: 2021-01-01T10:00:00-07:00
slug: autopkg-https
description: A script that helps AutoPkg recipe authors use HTTPS in download recipes, and context about why using HTTPS is important.
tags:
    - autopkg
    - macadmin
    - python
    - security
---

[AutoPkg](https://github.com/autopkg/autopkg) recipes automate and codify the often tedious tasks involved in packaging and distributing Mac software. Central to AutoPkg's greatness are the many built-in security measures that verify you're getting the software you intend — including [code signature verification](https://github.com/autopkg/autopkg/wiki/Using-CodeSignatureVerification), [embedded trust information in overrides](https://github.com/autopkg/autopkg/wiki/AutoPkg-and-recipe-parent-trust-info), and the `autopkg audit` command.

AutoPkg recipe authors should also follow another important security practice: use HTTPS URLs instead of HTTP whenever possible. Whether downloading actual software or downloading metadata about the software, using an HTTPS URL helps prevent [person-in-the-middle attacks](https://en.wikipedia.org/wiki/Man-in-the-middle_attack) and keep your organization's software pipeline secure.

In particular, the arguments and input variables used by the [URLDownloader](https://github.com/autopkg/autopkg/wiki/Processor-URLDownloader), [URLTextSearcher](https://github.com/autopkg/autopkg/wiki/Processor-URLTextSearcher), and [SparkleUpdateInfoProvider](https://github.com/autopkg/autopkg/wiki/Processor-SparkleUpdateInfoProvider) processors should use HTTPS if the option is available, and recipe authors should perform periodic checks to detect when software developers (or their CDNs) begin offering HTTPS downloads.

The security benefits aren't just theoretical; a few years ago, security researchers [demonstrated an attack](https://vulnsec.com/2016/osx-apps-vulnerabilities/) targeting Mac apps using insecure [Sparkle](https://sparkle-project.org/) feeds. Ben Toms wrote [a good article](https://macmule.com/2016/01/31/sparkle-updater-framework-http-man-in-the-middle-vulnerability/) detailing the Mac admin community's response to the vulnerability.

## HTTPS Spotter

Checking for the existence of HTTPS URLs can be tedious if you manage more than a handful of AutoPkg recipes, so I've written a Python tool called **HTTPS Spotter** that will automate the process for you. The [source code](https://gist.github.com/homebysix/66d1c8772baf5f731bb8ddf263f33401) is on GitHub and embedded below.

### Requirements

To use the script, you'll need Git and AutoPkg installed.

### Steps

1. Clone the script to your Mac (substitute the path to your source, if not _~/Developer_).

        git clone https://gist.github.com/66d1c8772baf5f731bb8ddf263f33401.git ~/Developer/https_spotter

2. Run the script with `--help` to see usage information.

        /usr/local/autopkg/python ~/Developer/https_spotter/https_spotter.py --help

3. Now run the script again, pointing it to your repository of AutoPkg recipes:

        /usr/local/autopkg/python ~/Developer/https_spotter/https_spotter.py ~/Developer/your-autopkg-recipes

    You'll see output that might look like this:

        ../homebysix-recipes/NeoFinder/NeoFinder.download.recipe
         Replace: http://www.cdfinder.de/en/downloads.html
            With: https://www.cdfinder.de/en/downloads.html
        ../homebysix-recipes/FontFinagler/FontFinagler.download.recipe
         Replace: http://www.markdouma.com/fontfinagler/version.xml
            With: https://www.markdouma.com/fontfinagler/version.xml

        2 suggested changes. To apply, run again with --auto.

4. Run the script again with the `--auto` flag in order to automatically apply the changes, or apply the changes manually in your preferred text editor.

5. Test the modified recipes prior to committing/pushing the changes to your public repo on GitHub.

    {{< admonition tip tip >}}
    Here's a one-liner that will run recently-modified recipes in "check only" mode:

    <pre>find * -iname "*.recipe" -mtime -1 -exec autopkg run -vvcq "{}" '+'</pre>
    {{< /admonition >}}

### Source code

The script is below. Suggestions or improvements are welcome!

{{< gist homebysix 66d1c8772baf5f731bb8ddf263f33401 >}}
