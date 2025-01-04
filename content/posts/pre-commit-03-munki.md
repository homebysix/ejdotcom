---
title: "Using pre-commit with Munki repos"
date: 2025-01-03T15:24:39-0800
slug: pre-commit-03-munki
description: How to use my shared pre-commit hooks for validating the contents of your Munki repository.
tags:
    - munki
    - git
    - macadmin
    - pre-commit
    - python
---

In previous posts, I [demonstrated the basics of pre-commit](../pre-commit-01-intro) and [recommended its use for linting AutoPkg recipes](../pre-commit-02-autopkg). This time, I'll discuss how pre-commit can be a lifesaver for those who use [Munki](https://www.munki.org/munki/) to deploy software.

## Setup

Setting up pre-commit in your Munki repo is simple:

1. [Install pre-commit](https://pre-commit.com/#install). I choose to do this with Homebrew:

    ```sh
    brew install pre-commit
    ```

2. Create, commit, and push a _.pre-commit-config.yaml_ file at the root of your Munki repository. To start, the contents of the file can be the following:

    ```yaml {linenos=table}
    repos:
      - repo: https://github.com/homebysix/pre-commit-macadmin
        rev: v1.18.0
        hooks:
          - id: check-munki-pkgsinfo
    ```

3. Activate the hooks in your Git repo:

    ```sh
    cd /path/to/your_munki_repo
    pre-commit install
    ```

First I'll cover what you get by adding the `check-munki-pkgsinfo` hook, then I'll show you how to customize it for your organization's needs.

## Check pkginfo files

{{% mark %}}The `check-munki-pkgsinfo` hook includes many sanity checks that are generally applicable to Munki pkginfo files.{{% /mark %}} Some of these checks are inherited from [others' Munki linting scripts](https://gist.github.com/bruienne/9baa958ec6dbe8f09d94), and others were born from my own past mistakes and learnings.

<!-- NOTE: Periodically verify and update this list of pre-commit checks. -->

As of this writing, the `check-munki-pkgsinfo` hook includes these checks:

- **The pkginfo file must be a valid plist.**

- **The pkginfo must contain certain required keys.** By default the required keys are `name` and `description`..

- **The key types must be what Munki expects.** The list of pkginfo keys Munki supports is [here](https://github.com/munki/munki/wiki/Supported-Pkginfo-Keys/); any keys that aren't the expected type will produce an error.

- **No duplicate imports should be present in the repo.** Duplicate imports are determined by the presence of a `__N` suffix (for example, `Firefox-81.0.1__1.plist`). See [below](#duplicate-import-warnings) for information on how to make this a warning instead of an error.

- **The values of certain keys must be valid.** For example, the `RestartAction` key must be either `RequireRestart`, `RequireShutdown`, or `RequireLogout`. Any other value will produce an error.

- **Specific typos in key names will generate errors.** For example, `minimum_os_vers` and `min_os_version` are both incorrect variations of the correct key, `minimum_os_version`.

- **Uninstall scripts must not be missing.** If the pkginfo specifies an `uninstall_method` of `uninstall_script`, there must also be an `uninstall_script` key with a script defined.

- **Items in the `items_to_copy` list must omit trailing slashes.**

- **Scripts must start with a shebang.**

- **Icons used by pkginfo files must be present in the repository.**

## Customizing pkginfo checks

Although the basic checks above may be good enough for many organizations, customization is available if you'd like to fine-tune or personalize the checks.

In general, pre-commit hooks are customized by [passing a list of arguments](https://pre-commit.com/#passing-arguments-to-hooks). These command-line arguments relay additional information to the check function.

Any valid yaml array will work for `args`. Three examples are shown below:

```yaml {linenos=table, hl_lines=[1,4,11]}
# single line
args: ["--some-things", "one", "two", "three", "--"]

# grouped lines
args: [
  "--some-things", "one", "two", "three",
  "--more-things", "four", "five", "six",
  "--"
]

# multiple lines
args:
- "--some-things"
- "one"
- "two"
- "three"
- "--"
```

{{< admonition tip "Tip: Quoting strings" >}}
I've chosen to quote all the examples on this page, because although YAML allows unquoted strings, it's difficult to troubleshoot errors resulting from bad quoting. Save yourself some time and quote by default.
{{< /admonition >}}

{{< admonition tip "Tip: Ending arguments" >}}
When including an <code>args</code> list with multi-value arguments, it's a good idea to ensure the final item in the array is <code>--</code>. This <a href="https://www.gnu.org/software/bash/manual/html_node/Shell-Builtin-Commands.html#:~:text=accepts%20%E2%80%98%2D%2D%E2%80%99%20to%20signify%20the%20end%20of%20the%20options">separator</a> tells the command-line interpreter that the list of custom arguments are complete. You'll see this separator used in all the examples in this post that have multi-part arguments.
{{< /admonition >}}

### Customizing required keys

If you want to specify pkginfo keys that should be required in addition to `name` and `description`, you can use the `--required-keys` argument, as shown here:

```yaml {linenos=table, hl_lines=[6]}
repos:
  - repo: https://github.com/homebysix/pre-commit-macadmin
    rev: v1.18.0
    hooks:
      - id: check-munki-pkgsinfo
        args: ["--required-keys", "name", "description", "category", "developer", "--"]
```

### Blocking applications for packages

If a `blocking_applications` array doesn't exist for a dmg installer, Munki can determine the proper blocking apps to use based on the dmg contents. However that's not true for pkg installers, so you may wish to require a `blocking_applications` array for pkg installers using this argument:

```yaml {linenos=table, hl_lines=[6]}
repos:
  - repo: https://github.com/homebysix/pre-commit-macadmin
    rev: v1.18.0
    hooks:
      - id: check-munki-pkgsinfo
        args: ["--require-pkg-blocking-apps"]
```

Note that an empty `<array/>` will satisfy this requirement, which Munki treats the same as if there is no `blocking_applications` key at all, for pkg installers.

### Missing icon warnings

By default, icons referenced by pkginfo files that are absent from your repository will generate errors. If you wish to ignore these errors, the following option will make them non-failing warnings instead:

```yaml {linenos=table, hl_lines=[6]}
repos:
  - repo: https://github.com/homebysix/pre-commit-macadmin
    rev: v1.18.0
    hooks:
      - id: check-munki-pkgsinfo
        args: ["--warn-on-missing-icons"]
```

### Duplicate import warnings

By default, files with the presence of a `__N` suffix (for example, `Firefox-81.0.1__1.plist`) in your repository will generate errors. However these may not actually be duplicates — perhaps they have different `supported_architectures` or other keys. If you wish to ignore these errors, the following option will make them non-failing warnings instead:

```yaml {linenos=table, hl_lines=[6]}
repos:
  - repo: https://github.com/homebysix/pre-commit-macadmin
    rev: v1.18.0
    hooks:
      - id: check-munki-pkgsinfo
        args: ["--warn-on-duplicate-imports"]
```

### Allowed catalogs and categories

Checking for specific allowable catalogs and categories is not enabled by default, but you can enable this check in your _.pre-commit-config.yaml_ file using the following arguments:

```yaml {linenos=table, hl_lines=[7,8]}
repos:
  - repo: https://github.com/homebysix/pre-commit-macadmin
    rev: v1.18.0
    hooks:
      - id: check-munki-pkgsinfo
        args: [
          "--catalogs", "testing", "stable",
          "--categories", "Creativity", "Engineering", "Productivity", "Security", "Utilities",
          "--"
        ]
```

### Valid shebangs

Scripts in Munki pkginfo files require a shebang in the first line in order to ensure they are interpreted by the proper scripting runtime. If you choose, you can define a pre-approved set of shebangs that you expect all scripts in your repo to use. Any shebangs not in this list will result in an error.

```yaml {linenos=table, hl_lines=["7-11"]}
repos:
  - repo: https://github.com/homebysix/pre-commit-macadmin
    rev: v1.18.0
    hooks:
      - id: check-munki-pkgsinfo
        args:
        - "--valid-shebangs"
        - "#!/bin/bash"
        - "#!/bin/sh"
        - "#!/usr/local/bin/managed_python3"
        - "#!/usr/local/munki/munki-python"
        - "--"
```

### Combining arguments

When combining arguments, just ensure the `--` argument is the last one in the list. A working _.pre-commit-config.yaml_ file that includes multiple of the above customizations may look like this:

```yaml {linenos=table, hl_lines=[7,12,15,21,26,27]}
repos:
  - repo: https://github.com/homebysix/pre-commit-macadmin
    rev: v1.18.0
    hooks:
      - id: check-munki-pkgsinfo
        args: [
          "--required-keys",
            "name",
            "description",
            "category",
            "developer",
          "--catalogs",
            "testing",
            "stable",
          "--categories",
            "Creativity",
            "Engineering",
            "Productivity",
            "Security",
            "Utilities",
          "--valid-shebangs",
            "#!/bin/bash",
            "#!/bin/sh",
            "#!/usr/local/bin/managed_python3",
            "#!/usr/local/munki/munki-python",
          "--warn-on-missing-icons",
          "--warn-on-duplicate-imports",
          "--"
        ]
```

## Check MunkiAdmin scripts

If you use [MunkiAdmin](https://github.com/hjuutilainen/munkiadmin), you may be excited to learn of its [presave/postsave scripts feature](https://speakerdeck.com/hjuutilainen/managing-munki-repositories-with-munkiadmin?slide=64), which can run scripts before or after saving files in your repository. The `check-munkiadmin-scripts` hook can be used to validate these scripts. This hook ensures that the scripts are executable and named properly.

Here's how you can add the `check-munkiadmin-scripts` hook to your _.pre-commit-config.yaml_ file:

```yaml {linenos=table, hl_lines=[5]}
repos:
  - repo: https://github.com/homebysix/pre-commit-macadmin
    rev: v1.18.0
    hooks:
      - id: check-munkiadmin-scripts
```

## Rebuild Munki catalogs

This last hook runs the `makecatalogs` command to ensure all referenced packages are present and catalogs are freshly built.

```yaml {linenos=table, hl_lines=[5]}
repos:
  - repo: https://github.com/homebysix/pre-commit-macadmin
    rev: v1.18.0
    hooks:
      - id: munki-makecatalogs
```

## Other helpful hooks

Many of the helpful hooks mentioned in my previous post about linting AutoPkg recipes are equally useful for Munki repos. Review the list [here](../pre-commit-02-autopkg#other-helpful-hooks).

## Conclusion

By integrating pre-commit hooks into your Munki repository, you can ensure that your pkginfo files and MunkiAdmin scripts are consistently validated and free of common errors. This not only helps maintain the integrity of your repository but also saves time and reduces the risk of deployment issues.

You can inspect the code that drives the [`check-munki-pkgsinfo`](https://github.com/homebysix/pre-commit-macadmin/blob/main/pre_commit_hooks/check_munki_pkgsinfo.py), [`check-munkiadmin-scripts`](https://github.com/homebysix/pre-commit-macadmin/blob/main/pre_commit_hooks/check_munkiadmin_scripts.py), and [`munki-makecatalogs`](https://github.com/homebysix/pre-commit-macadmin/blob/main/pre_commit_hooks/munki_makecatalogs.py) hooks on GitHub. If you have any questions or suggestions, I welcome issues and pull requests on the [pre-commit-macadmin](https://github.com/homebysix/pre-commit-macadmin) repository.
