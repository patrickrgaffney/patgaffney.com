+++
title = "Client Side Git Hooks - Part 2"
description = "Improving your version control workflow."
slug = "using-client-side-git-hooks-part-2"
date = 2018-03-05
draft = false
template = "post.html"
taxonomies.tags = ["git", "bash"]
+++

> This post was originally written for the [Monsanto Engineering Blog](http://engineering.monsanto.com/2018/03/05/using-client-side-git-hooks-2/), which is where I worked at the time.

[Last year I wrote](@/blog/2017-05-17-using-client-side-git-hooks.md) about how my team was using client-side [git hooks](https://git-scm.com/book/en/v2/Customizing-Git-Git-Hooks) to run sanity checks on our code before we commit. The example from last year was a pre-commit hook designed to perform static analysis on Javascript code using ESLint. One year later and we're still using them everyday, but we've made a few improvements to make the experience a whole lot smoother.

Let's walk through the problems we ran into over the past year, and the solutions we came up with. 

## The Missing Linter Problem

The previous version of this script had a `run_linter` function which directly invoked ESLint:

```sh
run_linter() {
  node node_modules/eslint/bin/eslint.js --fix --ext .js,.jsx src/ test/

  print_outro
}
```

What happens if you've recently performed the sacred `rm -rf node_modules` ritual? This is, after all, an [ancient tradition](https://twitter.com/iamdevloper/status/908335750797766656) for debugging Javascript issues.

Well, the `node` interpreter immediately throws an error because it cannot find the module you asked it to execute. It retreats with a positive exit code, so now you cannot commit until you `npm install` — assuming you gathered that from the error the `node` interpreter threw. Not the greatest experience.

![pre commit hook failure](/images/git-hooks-2.png)

We can write a simple function to ensure that the linter is installed before attempting to invoke it, and if not, present our colleague with the solution immediately. Bash has a series of [file test operators](https://www.gnu.org/software/bash/manual/html_node/Bash-Conditional-Expressions.html) that make this simple enough. While we're here, let's stop using `node` to interpret a `*.js` file in the node module directly, and instead invoke ESLint through the executable that `npm` already created for us in the [`.bin` directory](https://docs.npmjs.com/files/folders#executables).

```sh
test_for_linter() {
  if [ -x node_modules/.bin/eslint ]; then
    run_linter
  else
    echo '\033[1;31m𝙓 You do not have eslint installed!\033[0m'
    echo '\033[1;30m  Run "npm i" to install the linter.\033[0m'
    exit 1
  fi
}

run_linter() {
  ./node_modules/.bin/eslint --fix --ext .js,.jsx src/ test/ --cache

  print_outro
}
```

This solves our problem well enough, but using the relative path to execute ESLint still feels wrong. Well, if you've using `npm` 5.2 or greater, you might have noticed that an additional binary was installed — `npx`. `npx` allows you to run locally-installed packages without relative paths or `npm run <script>`. We can now safely eliminate that relative path.

```sh
run_linter() {
  npx -q eslint --fix --ext .js,.jsx src/ test/ --cache

  print_outro
}
```

The `-q` flag tells `npx` to suppress all of its output — this does not affect the invoked subcommand. Even if you're not using version 5.2 or greater of `npm`, you can still install the [standalone version](https://www.npmjs.com/package/npx) of `npx`.

## The Unstaged Changes Problem

I make a lot of small commits during development. It tends to make managing the historical logistics of a project easier — rebasing, squashing, and cherry-picking commits are all easier to perform with many small commits rather than a few large commits.

There are a great many pros to this method of source control — the one downside being that I often still have a *dirty* `git` status after staging my next commit. In other words, I'm editing 5 different files, but only 3 of these files are in the [staging area](https://git-scm.com/about/staging-area) ready to be committed.

![unstaged changes](/images/git-hooks-3.png)

The problem with having unstaged changes is they're not exempt from our pre-commit hook. If any of these files irritate the linter, we won't be able to commit. Obviously I only want the linter to analyze the files I'm about to commit, so let's [`stash`](https://git-scm.com/docs/git-stash) all files with changes not staged before we invoke the linter, then `pop` that same stash immediately after.

```sh
# Stash will be named "pre-commit-YYYY-MM-SS@HH:MM:SS"
STASH_NAME="pre-commit-$(date '+%Y-%m-%d@%H:%M:%S')"

stash_unchanged() {
  printf '\033[1;30m➡ Stashing unstaged changes as %s...\033[0m;\n' "$STASH_NAME"
  git stash push --quiet --keep-index --include-untracked "$STASH_NAME"
  run_linter
}

# ...run linter...

pop_unstaged() {
  git reset --hard --quiet && git stash pop --index --quiet
}
```

Lets walk through these two functions:

- `git stash push`: save the local modifications to a new stash entry.
  - `--quiet`: suppress all normal output
  - `--keep-index`: all changes already staged are left intact.
  - `--include-untracked`: all untracked (new) files are also stashed.
- `git reset --hard --quiet`: ensure we have a clean working directory.
- `git stash pop`: remove the last stash and apply it to the current working tree.
  - `--index`: re-instate the index's changes as well as a working tree's.
  - `--quiet`: suppress all normal output

To summarize, `stash_unchaged` stashes all files not currently in the staging area, and `pop_unstaged` pops that stash off the stack and back into the working directory. They operate as the inverse of each other. As a safety net, we give our stash a predictable name so that we can find it if trouble arises.

## The Missing Auto-Fix Problem

ESLint has a wonderful `--fix` flag that will fix any errors that it knows how to solve — most formatting errors can be fixed by just applying this flag. But given our current implementation, if the linter performs any auto-fixes, they won't be applied to our commit. Our files are already in the staging area, so any changes made to these files will not be automatically applied to the index — they must be added manually. To make matters more confusing, our previous `git reset --hard` (in `pop_unstaged`) would erase these changes before popping our stash.

We could manually add all changes made by ESLint to the staging area, but what if some random file had an error in it? Once again, we're only concerned with the files in this commit — we want this hook to operate under the assumption that all errors in other files will be fixed in the upcoming commits. For this, we need the `git update-index` command.

The `update-index` command is what the `git` authors would call a *plumbing* command (lower-level), as opposed to a *porcelain* command (user-friendly). Most documentation around `update-index` will inevitably steer you towards using `git add` — the more *user-friendly* command for updating the index. But, in this scenario, `update-index` has a `--again` flag that is a perfect solution: it updates the index **only** with unstaged changes made to files already in the index.

```sh
# ...stash_unchanged...

run_linter() {
  npx -q eslint --fix --ext .js,.jsx src/ test/ --cache
  git update-index --again
  
  pop_unstaged
}
```

Since we're now running multiple commands after `eslint`, we should probably merge the `run_linter` function with the `print_outro` function so we can properly inspect the exit code of ESLint.

```sh
# ...stash_unchanged...

run_linter() {
  if npx -q eslint --fix --ext .js,.jsx src/ test/ --cache; then
    echo '\033[1;32m✔ Eslint checks out! Pod bay doors opening...\033[0m'
    git update-index --again
    pop_unstaged
    exit 0
  else
    echo '\033[1;32m𝙓 Linter threw up! Pod bay doors sealed...\033[0m'
    pop_unstaged
    exit 1
  fi
}
```

## Putting It All Together

Our team uses client-side hooks like this every day. The `pre-commit` hook is really just the beginning, there are [over 15](https://git-scm.com/docs/githooks#_hooks) different `git` commands that have corresponding hooks.

The updated script is listed below.

```sh
STASH_NAME="pre-commit-$(date '+%Y-%m-%d@%H:%M:%S')"

print_intro() {
  user=$(git config --get user.name)
  printf '\033[1;34m➡ Time to pay the troll toll, %s...\033[0m\n' "$user"
  
  test_for_linter
}

test_for_linter() {
  if [ -x node_modules/.bin/eslint ]; then
    stash_unchanged
  else
    echo '\033[1;31m𝙓 You do not have eslint installed!\033[0m'
    echo '\033[1;30m  Run "npm i" to install the linter.\033[0m'
    exit 1
  fi
}

stash_unchanged() {
  printf '\033[1;30m➡ Stashing unstaged changes as %s...\033[0m;\n' "$STASH_NAME"
  git stash push --quiet --keep-index --include-untracked "$STASH_NAME"
  run_linter
}

run_linter() {
  if npx -q eslint --fix --ext .js,.jsx src/ test/ --cache; then
    echo '\033[1;32m✔ Eslint checks out! Pod bay doors opening...\033[0m'
    git update-index --again
    pop_unstaged
    exit 0
  else
    echo '\033[1;32m𝙓 Linter threw up! Pod bay doors sealed...\033[0m'
    pop_unstaged
    exit 1
  fi
}

pop_unstaged() {
  git reset --hard --quiet && git stash pop --index --quiet
}

print_intro
```
