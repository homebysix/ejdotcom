---
title: "Using Repo Lasso to Submit Pull Requests in Bulk"
date: 2024-06-23T16:30:00-0700
slug: bulk-prs-with-repo-lasso
description: Detailed example of using Repo Lasso to submit pull requests to a GitHub organization's repositories in bulk.
tags:
    - repo-lasso
    - git
    - macadmin
    - python
---

In 2021, I created a tool called [Repo Lasso](https://github.com/homebysix/repo-lasso) aiming to simplify the process of administering GitHub organizations with many similar repositories. Since then I've used the tool to submit hundreds of pull requests to [AutoPkg](https://github.com/autopkg/) recipe repositories.

I haven't said much publicly about Repo Lasso or how to use it. The time has come for me to submit another round of pull requests, and in this post I'll lead you through my process from beginning to end. Feel free to use this as a template for your own bulk pull request ideas.

## Contents  <!-- omit in toc -->

- [What is Repo Lasso?](#what-is-repo-lasso)
- [Setting up Repo Lasso](#setting-up-repo-lasso)
- [Syncing all org repos](#syncing-all-org-repos)
- [Creating a branch](#creating-a-branch)
- [Making changes](#making-changes)
- [Checking the changes](#checking-the-changes)
- [Committing the changes](#committing-the-changes)
- [Customizing and submitting the pull request](#customizing-and-submitting-the-pull-request)
- [Syncing and reporting](#syncing-and-reporting)

## What is Repo Lasso?

{{% mark %}}Contributors to GitHub organizations that have many *similar* repositories can use Repo Lasso to submit multiple pull requests for the same purpose across the entire organization.{{% /mark %}}

This could be useful when you're trying to influence conventions or style of the org's repos, but when ultimately it's the individual repo owners that will need to adopt your new convention or style.

This is also useful when applying existing agreed-upon community standards or best practices without having to force the changes with an administrative merge. For many communities (like AutoPkg) letting the repo owners have a chance to merge is the polite first step.

In this example, I'll be submitting another round of URL changes facilitated by my [HTTPS Spotter](https://www.elliotjordan.com/posts/autopkg-https/) script.

## Setting up Repo Lasso

Let's get started by making a local clone of Repo Lasso:

```sh
cd ~/Developer  # or wherever you store source code
git clone https://github.com/homebysix/repo-lasso.git
```

In order to install and maintain the Python requirements, I find it easiest to create a [virtual environment](https://pypi.org/project/virtualenv/):

```sh
cd repo-lasso
virtualenv venv
source venv/bin/activate
pip install -r requirements.txt
```

The pull requests I'll be submitting utilize [AutoPkg](https://github.com/autopkg/autopkg) and my [HTTPS Spotter](https://www.elliotjordan.com/posts/autopkg-https/) script, which I already have installed. You may have different requirements not included here, depending on the pull requests you'll be submitting.

For convenience, we can store our GitHub configuration in a *config.json* file within the Repo Lasso folder.

```sh
cat << EOF > config.json
{
    "github_username": "<your_username_here>",
    "github_token": "<your_token_here>",
    "github_org": "autopkg",
    "excluded_repos": [
        "autopkg",
        "index",
        "setup-autopkg-actions"
    ]
}
EOF
```

Passing the above configuration parameters in via the command line is also supported.

{{< admonition tip tip >}}
Use <code>./RepoLasso.py --help</code> at any point for more information on the available actions and options.
{{< /admonition >}}

## Syncing all org repos

Now we're ready to perform our first sync with all the repos in the organization. Kick off the sync with this command:

```sh
./RepoLasso.py sync
```

During this process, Repo Lasso will ask you if I want to make *forks* of the organization repos in your individual GitHub account. Type `y` to do this.

Repo Lasso will also ask if you want to make *clones* of the repos locally. Type `y` to do this.

For my organization of 150 or so repos, this took about 8 minutes. Subsequent syncs take less time because the fork/clone steps won't need to be done.

## Creating a branch

Before making any changes to the repos, we're going to create a new branch on all of our local clones using this command:

```sh
./RepoLasso.py branch 20240616-https
```

(To preserve chronological sorting, I like to prefix the branch name with today's date.)

This creates an "initiative" in Repo Lasso, as indicated by the pull request template saved to the file at *initiatives/20240616-https.md*. We'll come back to that file later.

## Making changes

Next I'll be making the needed changes to the repositories. My HTTPS Spotter script handles this part automatically by replacing HTTP software download URLs with their HTTPS equivalent whenever possible. (See [this previous post](../autopkg-https) for details.)

```sh
python3 ../https_spotter/https_spotter.py -v repos/autopkg --auto
```

For previous changes, I've employed other basic scripts or just performed a [multi-file find-and-replace](https://learn.microsoft.com/en-us/visualstudio/ide/finding-and-replacing-text?view=vs-2022#multifile) within a text editor.

{{< admonition note note >}}
Make sure all the changes you make on this branch align with the same "initiative." If you need to submit multiple unrelated changes, it may be better to do separate branches/pull requests for each.
{{< /admonition >}}

## Checking the changes

Repo Lasso now has a `check` feature that allows you to run a script against the modified files and optionally revert them to their unmodified state if the script doesn't exit successfully.

Here's a simple script that tests changed AutoPkg recipes.

```sh
#!/bin/bash

# $1 is the path to the git clone
# $2 is the path to the file being tested
# $3 is the number of the current attempt (starting with 0)

CACHE_DIR="/tmp/repo-lasso/autopkg-cache"

if [[ $3 -eq 0 ]]; then
    # Clear cache before first attempt of each check
    rm -rf "$CACHE_DIR"
fi

if [[ $2 =~ \.download\.recipe ]]; then
    # Allow download recipes to run
    autopkg run --quiet --key CACHE_DIR="$CACHE_DIR" "${1}/${2}"
elif [[ $2 =~ \.pkg\.recipe ]]; then
    # Allow pkg recipes to run
    autopkg run --quiet --key CACHE_DIR="$CACHE_DIR" "${1}/${2}"
else
    # Limit other recipes to check-only
    autopkg run --quiet --key CACHE_DIR="$CACHE_DIR" --check-only "${1}/${2}"
fi
```

Repo Lasso will capture the exit codes of modified recipes, and if the exit code differs before and after, I can use `--revert` to undo the change.

```sh
./RepoLasso.py check checks/autopkg_run.sh --revert --tries 3
```

I've also specified that I want three tries before and after, just to make sure that successes are repeatable.

## Committing the changes

With the remaining changes tested successfully, we're now confident that we can commit and push them into a new pull request.

```sh
./RepoLasso.py commit "Switch to HTTPS URLs whenever possible"
```

This saves the changes with a consistent commit message across all locally modified clones.

## Customizing and submitting the pull request

Now we can return to the pull request template located at *‌ initiatives/20220804-https.md*. This is the title and message that repo owners will see on GitHub. It's a good idea to provide context for why this pull request is necessary, along with information about any testing performed, in order to make it easy to merge.

```md
# Update all available URLs from HTTP to HTTPS

It's important to use HTTPS for downloading software whenever possible in order to avoid the
possibility of person-in-the-middle attacks (like the one that affected the Sparkle update
framework [in January 2016](https://vulnsec.com/2016/osx-apps-vulnerabilities/)).

For this pull request, I detected and changed to HTTPS URLs automatically using my
[HTTPS Spotter](https://www.elliotjordan.com/posts/autopkg-https/) script, and then tested all
changed recipes manually to ensure exit codes of recipe run remains the same before/after the
change.

This PR was submitted using [Repo Lasso](https://github.com/homebysix/repo-lasso).

Thanks for considering!
```

Once the template is customized, we can submit the pull request using this command:

```sh
./RepoLasso.py pr
```

As Repo Lasso processes each pull request, the URL to the PR will be shown in the output. I like to Command-double-click on each of these URLs as they appear to verify that they all look good.

Now that we're done submitting PRs, we can switch back to the default branch on our local clones:

```sh
./RepoLasso.py reset
```

If you have additional pull requests to submit, you can repeat the branch, change, check, commit, and PR steps again for each pull request.

## Syncing and reporting

I periodically like to keep all my local clones synced with upstream changes. You can run this command again to do that:

```sh
./RepoLasso.py sync
```

Repo Lasso includes a convenient reporting feature that lets you keep track of the status of each of the pull requests you've opened. You can generate this report with:

```sh
./RepoLasso.py report
```

The report is produced in Markdown format, and looks like this when rendered:

![Repo Lasso Report](../images/repo-lasso-report.png)

If any pull requests show status "conflict," you can click to view them in a browser and see what action you can take to help the repo authors resolve.

---

I hope this has made Repo Lasso's usefulness a bit more concrete. Although I wrote Repo Lasso for myself and am likely its sole user, I hope GitHub admins who manage multiple similar repositories in an organization can eventually benefit from the tool as well.
