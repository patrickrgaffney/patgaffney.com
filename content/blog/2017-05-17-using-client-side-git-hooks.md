+++
title = "Using Client-Side Git Hooks"
description = "Ensure your code matches your team's expections."
slug = "using-client-side-git-hooks"
date = 2017-05-17
draft = false
template = "post.html"
taxonomies.tags = ["git", "bash"]
+++

> This post was originally written for the [Monsanto Engineering Blog](http://engineering.monsanto.com/2017/05/17/using-client-side-git-hooks/), which is where I worked at the time.

[Git hooks][githooks] are a simple solution to an ongoing problem &mdash; sometimes we push ugly, breaking code. Sometimes we get wrapped up in the problem we're solving and forget that other humans are going to have to read and understand this in the future. Our fellow developers are nice people, they shouldn't be subjected to our forgetfulness.

[githooks]: https://git-scm.com/book/en/v2/Customizing-Git-Git-Hooks

Often our solutions to this problem are overtly complex. We *could* set up our test suite to run every time we save a file, but that would get out of hand fairly quickly. We *could* let our editors paint code red as we type expressions our linter finds unsatisfactory, but do we really want our editor to yell at us? We *could* set up elaborate events to trigger on some automation server that runs the test suite and lints the code &mdash; but why should we catch this problem only after our code has been committed to history and deployed?

Git hooks aim to automate this problem. Hooks are just scripts to be triggered when important events occur. When a specific `git` command is issued, a check is made to determine if there is an associated script to run. Hooks can be written for the client or the server, but we're going to focus on the client here.

## Sanity Checks

Let's say we're writing a [React][react] front-end for a web application and we're linting our code with [ESLint][eslint]. In the age of JSX, it's important for our team that we maintain a consistant style for our rendered markup. To sanity-check the code we've written, we can write a `pre-commit` hook to be triggered everytime someone on our team issues `git commit`.

[react]: https://facebook.github.io/react/
[eslint]: http://eslint.org

```bash
#!/bin/bash

function print_intro {
  user=$(git config --get user.name)
  echo '\033[1;34m➡ Time to pay the troll toll, ' "$user" '...\033[0m'

  run_linter
}

function run_linter {
  node node_modules/eslint/bin/eslint.js --fix --ext .js,.jsx src/ test/

  print_outro
}

function print_outro {
  if [ $? == 0 ]; then
    echo '\033[1;32m✔︎ Linter checks out! Pod bay doors opening...\033[0m'
    exit 0
  else
    echo '\033[1;31m𝙓 Linter threw up! Pod bay doors forever sealed...\033[0m'
    exit 1
  fi
}

print_intro
```

If you're hazy on your shell scripting, don't worry &mdash; there's very little going on here. First, we grab the user's name from their `git config`. A small alert is printed, and we tell our linter to run. Finally, in `print_outro` we determine if the linter exited successfully via `$?`. If it exited with errors &mdash; a non-zero status &mdash; our script shares in the disappointment and returns a general error status of `1` to the calling shell.

The important thing to note here is that when the linter exited gracefully, we followed suit. We only exit with an error if the linter did. This is how virtually all git hooks operate. The `pre-commit` hook is fired before the user even types a commit message. It has full access to the snapshot of the repository at that moment. Exiting from this hook with a non-zero status will **abort the commit** &mdash; effectively preventing the user from commiting their changes.

This can be frustrating, so it's important to alert the user that something went wrong. We also write all of the linter's output to `stdout` so the errors can be easily detected. `git` leaves the staging area unchanged, so any errors can be fixed and another commit can be issued in quick succession.

There are client-side hooks for almost [all of the events][events] `git` triggers &mdash; `pre-commit` is just the tip of the iceberg.

[events]: https://git-scm.com/docs/githooks

![pre commit hook failure](/images/git-hooks.png)

## Hook Installation

The only trouble with client-side hooks is installing them. `git` looks for hooks in the `hooks` subdirectory of the `$GITDIR` &mdash; by default the relative path is `.git/hooks/`. When a new repository is initialized, the hooks directory is filled with a series of sample hooks that are turned off by default. Any **executable** file in the `hooks/` directory that is named for the event it should be triggered against will be picked up by `git`.

In order to keep all developers on our team up-to-date with the proper hooks, we've found it best to check them into `git`. We have a `hooks/` directory at the root of all our repositories where hooks reside. We use a simple installation script to create symbolic links between the version-controlled hooks in this directory and their home in `.git/hooks/`:

```bash
#!/bin/bash

# Symlink all the githooks.
# ln -s :: create symbolic link
# ln -v :: verbose mode- print detail
# ln -f :: overwrite target file if it exists

GITDIR="../.git/hooks"
CURDIR=$(basename "$PWD")

echo '\033[1;34m➡ Installing all the githooks...\033[0m'

if [[ $CURDIR != "hooks" ]]; then
  echo '\033[1;31m✘ You need to execute this command from the hooks/ directory\033[0m'
  exit 1
fi

# Create symlink for each githook into $GITDIR
for file in $(ls -A)
do
  if [[ $file == "installhooks.sh" ]]; then
    continue
  elif [[ -x $file ]]; then
    ln -svf "$PWD/$file" "$GITDIR"
  else echo '\033[1;34m➡ Skipping' "$file" 'because you cannot execute it\033[0m'
  fi
done

echo '\033[1;32m✔︎ Githooks successfully installed...\033[0m'
```

Once again, this shell script is fairly simple. It first checks to ensure you're issuing it from the `hooks/` directory (this is important for the symlink), then loops through the files, installing those whose [executable bit][fileperms] is flipped.

Because all the hooks are checked into `git`, any changes made to a hook are circulated to all developers on our team the next time they run `git pull`. The symlinks take care of the rest &mdash; ensuring that the hooks reside where `git` will find them.

[fileperms]: https://en.wikipedia.org/wiki/File_system_permissions#Permissions

If for some reason shell scripts aren't your thing, no worries, you can write your hooks in any language that will exit with a Unix status code.

## Popular Amongst Friends

Client-side hooks are ideal for these types of repetitive tasks: linters, tests, formatting merge commits, and so on. They provide a benefit best served over time &mdash; our team's `pre-commit` hooks keep our code in a consistent, working state.

> The script written in this post was improved upon in a [second post](@/blog/2018-03-05-using-client-side-git-hooks-2.md).
