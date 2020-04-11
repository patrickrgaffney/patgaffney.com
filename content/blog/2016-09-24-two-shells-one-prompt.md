+++
title = "Two Shells and A Prompt"
description = "Lessons learned from ditching zsh, coming home to bash, and writing a prompt."
slug = "two-shells-one-prompt"
date = 2016-09-24
draft = false
template = "post.html"
taxonomies.tags = ["bash", "zsh", "git"]
+++

I recently made the switch back to [bash][bash] after a long spell with [zsh][zsh]. I'm a little conflicted, but there were three driving factors in my breakup with zsh:

1. Zsh can be pretty slow sometimes &mdash; especially when you add [oh-my-zsh][omz] to the mix.
2. Bash is everywhere.
3. Virtually all shell scripts are written in bash.

Lets start with the first point. [A lot][zsh-slow] of people [have written][zsh-slow2] about why they think zsh is slow. [Most of the time][zsh-slow3] it's a combination of oh-my-zsh and overweight startup files. I could use zsh without oh-my-zsh, but that seems unreasonable. If I disable oh-my-zsh, I am going to end up rebuilding a lot of its functionality &mdash; and it will probably be half-assed in comparison. [^zsh-alts] And it's not just the startup time, I often find myself *waiting* on the completions after triggering them. Is this worth the in-buffer command highlighting?

Point number two: bash is literally everywhere. Bash is already on both of the servers I pay for. When I am am forced to use a computer at my college &mdash; *first I find a Mac* &mdash; bash is already there when I launch the terminal. When I have to `ssh` into the universities Linux servers to turn in projects, I am greeted with a friendly `bash4.2$` prompt. Bash is the air we breathe, it's the dirt beneath our feet. It's practically a given that it will be on any UNIX system you can find *and* it will be the default shell. Even Windows [has bash now][windows]!

Bash is also the standard for writing shell scripts. If you reach a point where you or an application running needs to execute a series of commands, there is a 95% chance that it is a bash script. This doesn't necessarily mean I have to break-up with zsh, there is actually a lot of interoperability between the two [^fish]. The problem is that I often find myself needing to change some part or parts of these scripts. It usually goes something like this:

- Open the script in TextMate.
- Slowly search for the command that needs to be changed or updated.
- Cower in fear at the foreign syntax.
- Head for the hills, never looking back.

This is no good. So my decision is made for me &mdash; if I am worth my weight in salt as a programmer, I need to learn bash. I need to be *proficient* with bash.

## bash and macOS

When you launch the Terminal on <del>OS X</del> macOS, you are greeted with a bash prompt. The problem is, this software is roughly 10 years old. For reasons having to do with [licensing][license], Apple continues to ship version 3.2 with macOS. Meanwhile bash marches on &mdash; version 4.4 was released a week ago.

Luckily this is a solvable problem. [Homebrew][brew] exists, it's awesome, and it's easy:

```sh
# update all homebrew formulae
brew update

brew install bash

# change the the default shell to bash
sudo chsh -s /usr/local/bash
```

Alright, now we have bash 4.4. But we have a long way to go before this is a replacement for zsh. For starters, the default prompt needs some surgery.

## Bash: A Graphic Novel

When I decided to venture out into these uncharted waters, the first thing I did was search for a reference manual for bash. What do you know, one of those exists and it's written by the current and long time maintainer [Chet Ramey][chet]. You can find it on a [single webpage][bash-ref-html], a [single PDF][bash-ref-pdf], or a [series of bounded dead tress][bash-ref-trees] [^trees]. I opted for the dead trees, because I am a heartless bastard who loves highlighting things and reaching for books when I want answers.

Over a string of weekends I read this reference manual. This shouldn't come as a surprise to anyone, but it is **not** a particularly compelling read. It's called *Bash 4.3: Reference Manual* after all, not *Bash 4.3: A Graphic Novel*. Nonetheless it was enlightening. Chapter 6 of this manual deals with features specific to bash, and this is where I'll begin.

## A Brief Tour Through the Bash Startup Files

Bash can be *invoked* in many different ways, but all you really need to know is there are two different invocations:

1. As an **interactive shell**.
2. As an **non-interactive shell**.

Because I set bash to be my default shell, it is *invoked* when I launch the terminal. This effectively makes it an interactive shell. Alternatively, it could be invoked as a non-interactive shell. The difference appears to be mostly semantics in our GUI-based world, but depending on how bash is invoked will determine which startup files are executed. Ahh, startup files. Those are just shell scripts that are executed at the time bash is invoked. They usually tell bash to enable or disable certain features, but they can execute any arbitrary command.

Here is a brief tour through the bash startup file landscape:

- If invoked as an **interactive login shell**, it reads and executes commands from `/etc/profile` if it exists, then executes commands from the first file in the following list that exists and is readable:
    1. `~/.bash_profile`
    2. `~/.bash_login`
    3. `~/.profile`
- If invoked as an **interactive non-login shell** (as a subshell), it reads and executes commands from `~/.bashrc`.
- If invoked **non-interactively** (to run a shell script), it looks for the `BASH_ENV` environment variable, and if found it's value is expanded and used as the name of a file to read and execute.
- If invoked **with `sh`**, it reads and executes commands from `/etc/profile` if it exists, then attempts to read and execute commands from `~/.profile`.
- If invoked **in POSIX mode** (with the `--posix` option), it looks for the `ENV` environment variable, and if found it's value is expanded and used as the name of a file to read and execute.
- If invoked **by a remote shell daemon** (`sshd`), it attempts to read and execute commands from `~/.bashrc`.
- If invoked with **the *effective* user group id not equal to the *real* user group id**, no startup files are read.

This is where I would normally run for the hills.

Unfortunately, all these different ways of invoking bash creates uncertainty. What I want is a surefire solution to this problem &mdash; I want one file that I can guarantee will be executed every single goddamn time bash is invoked.

I should note that `/etc/profile` is probably not a great place to put any startup commands. If you don't have `sudo` access, then you probably can't edit it anyway. This file is system-wide, so it will effect anyone with an account on your machine. On my MacBook, this file only has two commands. It first executes the `path_helper` binary, then makes a call to `/etc/bashrc`, if it exists. Once again, because these files are system wide, they don't pose a good solution to my problem.

My solution is to create three different files. The first two, `~/.bash_profile` and `~/.profile`, contain exactly one logical expression:

```sh
# if .bashrc exists, load it
if [ -f ~/.bashrc ]; then
  . ~/.bashrc;
fi
```

That's right, I'm passing the buck. I am going to put all of the commands I want executed at startup in one file: `~/.bashrc`. I choose `~/.bashrc` because if we launch an *interactive non-login shell*, it is our only option. This takes care of four out of six of the above situations (interactive login, interactive non-login, with `sh`, by remote daemon). Thats 66%, not to shabby.

I am not going to concern myself with bash launching itself in *POSIX mode*, as this is something I would half to do manually, and I can just pretend it doesn't exist. Ignorance is bliss.

That leaves one final scenario: bash launching *non-interactively*. I am going to punt on this scenario too &mdash; I figure I don't need to worry about how the shell is set up in an environment where I don't interact with it.

## Settings Galore

Before I go all ANSI-Color-Codes on the ridiculous default prompt, lets modify some of the shells behavior. Chapter 4 of the *Reference Manual* deals with the shell commands built into bash. Two of these commands &mdash; `set` and `shopt` &mdash; change the values of shell options, thus enabling and disabling specific features.

### The `set` Builtin

In true UNIX fashion, you can use `set` in two different ways:

1. The *readable* way: Use `set -o <option-name>` to toggle settings on or off.
2. The *unreadable* way: Use `set -x` to turn settings on, `set +x` to turn settings off.
    - Where `x` is a single ASCII character that corresponds to a specific `option-name`.

I'm going to use the first option for this, because I'm not an animal.

Bash comes with about half of the `set` options turned on by default, and they're fairly sensible choices. There are two additional settings I have decided to toggle. The first, `ignoreeof`, prevents an interactive shell from exiting upon reading `EOF`. This is infinitely useful if you have ever written very buggy C code [^c-code] &mdash; you end up pounding on Control-D until your keyboard breaks. The second, `notify`, prints the status of completed background jobs immediately, instead of waiting for the next prompt.

Finally, we actually have something in our `.bashrc`:

```sh
# Do not exit an interactive shell upon reading EOF.
set -o ignoreeof;

# Report the status of terminated background jobs immediately, 
# rather than before printing the next primary prompt.
set -o notify;
```

### The `shopt` Builtin

I like to pronounce `shopt` like I assume someone from Alabama would pronounce the past-tense of "shopping." This is the real meat and potatoes. Almost all of the options available to `shopt` are turned off by default, which makes it a goldmine. Basic usage is `shopt [-su] <optname>`, where `s` sets the option and `u` unsets the option.

Because there are so many of theses, I am just going to list the ones I toggled on, as they appear in my `.bashrc` file. But, at minimum, everyone should really turn on the ones that correct your typos. I have also omitted the settings that are on by default.

```sh
# Executed a directory name as if it were an argument to cd.
shopt -s autocd

# Correct spelling errors in directory names given to cd.
shopt -s cdspell

# Check the hash table for a command name before searching $PATH.
shopt -s checkhash

# Update the window size variables after each command.
shopt -s checkwinsize

# Save all lines of a multi-line command in the same history entry.
shopt -s cmdhist

# Correct spelling errors on directory names during word completion.
shopt -s dirspell

# Enable extended pattern matching features.
shopt -s extglob

# Enable `**` pattern in filename expansion to match all files,
# directories and subdirectories.
shopt -s globstar

# Append the history list to $HISTFILE instead of replacing it.
shopt -s histappend

# Save multi-line commands to the history with embedded newlines
# instead of semicolons -- requries cmdhist to be on.
shopt -s lithist

# Do not attempt completions on an empty line.
shopt -s no_empty_cmd_completion

# Case-insensitive filename matching in filename expansion.
shopt -s nocaseglob

# Make echo builtin expand backslash-escape-sequence.
shopt -s xpg_echo
```

### Useful Shell Variables

Certain bash shell variables supplement the `shopt` settings, or add additional functionality. All of the variables are listed in Chapter 5 of the *Reference Manual*. If you decide to utilize any of these, ensure they are exported to override any default values.

The history list can be controlled via a few different variables.

```sh
# History file control:
#   - ignorespace = don't save lines that begin with a space
#   - ignoredups  = don't save duplicate lines
export HISTCONTROL='ignorespace:ignoredups'

# Maximum number of lines/commands to save in the history file.
export HISTFILESIZE=150
export HISTSIZE=150
```

Bash ships with `force_ignore` turned on by default. It uses the shell variable `FIGNORE` to list suffixes to be ignored when performing word completion. Similar functionality can be obtained for filename expansion by supplying a list of filename patterns to `GLOBIGNORE`. Both require the list to be colon-separated, much like the `PATH`.

```sh
# Ignore files with these suffixes when performing completion.
export FIGNORE='.o:.pyc'

# Ignore files that match these patterns when 
# performing filename expansion.
export GLOBIGNORE='.DS_Store:*.o:*.pyc'
```

Finally, lets add color to the output of the `ls` command. There is a caveat however &mdash; this is mostly OS-dependent. In other words, because `ls` is not built into bash, it is included with the operating system and therefore differs depending on who wrote the utility. I am going to cover the `ls` included on macOS [^bsd].

To enable colored output on `ls`, you need to do either of these two things:

- Supply the `-G` option to `ls` every time you use it (or set an alias).
- Set the variable `CLICOLOR` to a non-zero integer.

Since either of these options will work, I'll just set the `CLICOLOR` variable so I can move on with my life.

Just telling `ls` to use colors is not enough though, you need to tell it *which* colors to use [^ls]. This is done by setting the `LSCOLORS` variable to a ridiculous string of characters, each of which corresponds to a file type, foreground color, and background color.

This sounds really convoluted &mdash; and it is &mdash; so my advice would be to use an [excellent tool made by Geoff Greer][ls-colors] to generate this string for you. My settings are below.

```sh
# Set colors for ls command.
#   1.  directory: ex
#   2.  symbolic link: fx
#   3.  socket: gx
#   4.  pipe: bx
#   5.  executable: cx
#   6.  block special: aH
#   7.  character special: aA
#   8.  executable with setuid bit set: cA
#   9.  executable with setgid bit set: cH
#   10. directory writable to others, with sticky bit: eA
#   11. directory writable to others, without sticky bit: eH
export CLICOLOR=1
export LSCOLOR='exfxgxbxcxaHaAcAcHeAeH'
```

## A Brief Tour Through the ANSI Color Codes

With oh-my-zsh I had an incredibly informative prompt, and I'm not ready to live in a world where I stare at `bash4.4$` all day. But before we start adding random special characters to our prompt string, it's useful to understand how the shell displays colors.

Virtually all text terminals use [ANSI Escape Codes][ansi] to control the color and formatting of strings written to the terminal. This system of using escape codes is roughly 50 years old now, so it's not exactly intuitive. None of this is made easier by the string syntax in bash.

Strings in bash come in four different forms:

- **Single quotes**: all characters are represented by their literal values.
- **Double quotes**: all characters are represented by their literal values except `$`, `'`, `\`, and `!` if history expansion is enabled.
- **C-style strings**: all characters are represented by their literal values except for backslash-escaped characters as specified by the ANSI C Standard.
- **Locale-specific strings**: all characters are translated according to the current locale.

```sh
# single quoted string
'a string'

# double quoted string
"!ls"

# c-style string -- notice the `$`
$'\ttab then newline\n'

# locale-specific string -- notice the `$`
$"translate me"
```

Because the color codes are escape sequences, it seems obvious we should use C-style strings. This gets tricky because the color codes are not part of the ANSI C Standard. Most [popular prompts][git-prompt] get around this by using the octal code for the [ASCII Escape character][ascii], `\033`. While this works, you end up *escaping* the escape sequence &mdash; it makes for a huge mess of a string. Luckily, bash provides the `\e` or `\E` sequence to print an escape character.

Back to the color codes, they take the basic form:

```sh
# \e -- escape sequence that signals an escape character
# {parameter;} -- a parameter to describe the formatting and color
$'\e[{parameter;}{parameter}m'
```

The sequence begins with `\e[`, followed by parameters. Naturally, the order of the parameters is arbitrary. The first parameter is required, but it only needs a semicolon if there is a second parameter. The second parameter is optional, but it gets a semicolon if there is an optional third parameter, and so on. The end of the sequence is marked with the letter `m`.

But what the hell is a parameter? Well, it's really just an integer code. There's [quite a few options][ansi-colors], the most common are:

- `0`: reset
- `1`: bold text
- `30`: black text
- `31`: red text
- `32`: green text
- `33`: yellow text
- `34`: blue text
- `35`: magenta text
- `36`: cyan text
- `37`: white text
- `39`: default text color
- `40`: black background
- `41`: red background
- `42`: green background
- `43`: yellow background
- `44`: blue background
- `45`: magenta background
- `46`: cyan background
- `47`: white background
- `49`: default background color

So, if you're still with me, to make `string` have bold red text, your bash string would be `$'\e[1;31mstring'`. This example is a little misleading, because once we set the formatting parameters to be anything other than normal, we must reset them. If the formatting is not reset, then anytime you use that string in a parameter, variable, or command expansion all of the text past the point at which the expansion took place will use that formatting. So the correct bash string would be `$'\e[1;31mstring\e[0m'`

## Building a Better Prompt

The default bash prompt is `bash-4.4$`. Actually, this is just the *first* prompt string, there are four in total. Each prompt is set by the shell variable `PS#` where `#` is 1, 2, 3, or 4. `PS1` is the prompt printed after &mdash; or before, depending how you look at it &mdash; every command is executed.

### Prompt Escape Sequences

Bash has quite a few special characters which &mdash; when they appear in the prompt string &mdash; output generated text. The most useful are listed below:

- `\d`: The date, as "Weekday Month Day".
- `\e`: An escape character.
- `\h`: The hostname of the machine, up to the first `.`.
- `\H`: The hostname of the machine.
- `\n`: A newline.
- `\s`: The name of the shell.
- `\t`: The time, in 24-hour "HH:MM:SS" format.
- `\T`: The time, in 12-hour "HH:MM:SS" format.
- `\@`: The time, in 24-hour am/pm format.
- `\A`: The time, in 24-hour "HH:MM" format.
- `\u`: The username of the current user.
- `\v`: The version of bash.
- `\w`: The current working directory, with a `~` representing `$HOME`.
- `\w`: The basename of `$PWD`, with a `~` representing `$HOME`.
- `\#`: The command number of this command.

For example, that default bash prompt, `bash-4.4$`, would be set as:

```sh
PS1='\s-\v$'
```

### Making a Basic Prompt

To get the ball rolling, I want a prompt that at minimum tells me:

- The number of commands I have executed.
- My username.
- The hostname.
- The working directory.
- The time.

None of these are ridiculous requests. In fact, all of them have prompt escape sequences.

```sh
export PS1='\#. \u@\h in \w at \A\n$ '
```

On my local machine, this will give me the following prompt:

```
17. patrickrgaffney@patmac in ~/Code/dotfiles at 16:44
$ 
```

Not a bad start. My only concern is that once I get deep into some nested directory the first line is going to end up wrapping. I already added a newline in order to give myself room to actually type commands. 

Bash uses the shell variable `PROMPT_DIMTRIM` to fix this problem. If set to a positive integer, that value is used as the number of trailing directory components to retain when expanding the current working directory for the `\w` and `\W` prompt string escapes. Directories that are removed are replaced with an ellipsis.

I have it set to `3` in my `.bashrc`:

```sh
# Hide all but the 3 deepest directories for PS1.
export PROMPT_DIRTRIM=3

# Set the prompt.
export PS1='\#. \u@\h in \w at \A\n$ '
```

This fixes my nested directory issue:

```
3. patrickrgaffney@patmac in ~/.../.git/objects/info at 17:00
$
```

### Adding Some Color to the Prompt

We already know how the ANSI color codes work, so this should be no big deal. Truthfully, the hardest part was choosing which prompt escape sequence got which color. I also decided to split up the assignment of `PS1` into multiple assignments. Bash allows strings to be appended to each other with the `+=` operator. Also note the addition of `$` to the front of the strings in order to make them C-style strings.

```sh
# Begin appending information to PS1
export PS1=''

# Add cyan command-number: '\#'
PS1+=$'\e[1;36m\#.\e[0m'

# Add bold username: '\u'
PS1+=$' \e[1m\u\e[0m'

# Add bold-blue hostname: '\h'
PS1+=$'\e[1;34m@\h\e[0m in'

# Add bold-red working directory: '\w'
PS1+=$'\e[1;31m\w\e[0m'

# Add bold-green time: '\A'
PS1+=$'at \e[1;32m\A\e[0m'

# Add dollar-sign `$`
PS1+=$'\n$ '

# Hide all but the 3 deepest directories for PS1.
export PROMPT_DIRTRIM=3
```

Which gives me:

<pre><span style="font-weight: 700; color:#d08770;">25. </span><span style="font-weight: 700;">pat</span>@<span style="font-weight: 700; color:#8fa1b3;">patmac</span> in <span style="font-weight: 700; color:#bf616a;">~/Code/dotfiles</span> at <span style="font-weight: 700; color:#a3be8c;">17:11</span>
$</pre>

So far, so good.

### Displaying the Git Branch

I use [Git][git] for just about everything I do. It's completely transformed the way I work on projects and keep track of schoolwork. One of git's best features is the cheap branching. You can create a branch for anything, any time, and it's so inexpensive that it's invaluable.

Git comes with a [shell script][git-shell-script] that you can use to add branching information to your prompt. Frankly, it has a lot more functionality than I will ever use, or really care to learn to use. And considering I'm trying to familiarize myself with the bash syntax, I should probably try and roll my own solution.

If you run `git branch --help`, you'll find the `--no-color` option. It prints the the list of branches &mdash; with an asterisk next to the current branch &mdash; without using the ANSI color codes, even if they are turned on. This is probably the best way to get the data we need on the branches. Then it can be parsed to find which line has the `*` on it.

In order to use whatever solution I come up with in the prompt it has to be a function. Bash doesn't *really* have return values from functions &mdash; at least not in the same way other languages do. But, if you call a bash function from command substitution inside a string, that function will `echo` into the string.

The bash shell variable `IFS` contains a list of characters that separate fields when the shell executes an expansion. When used with the `for-in` loop, it concatenates the loop variable into separate strings. In other words, set `IFS` to the newline character and `for line in $string` iterates over the lines in `$string`. Then all we have to do is pattern match against the line that has the asterisk, remove the asterisk, and `echo` the branch name.

```sh
function git_branch {
    IFS=$'\n'
    local branches=$(git branch --no-color 2> /dev/null)
    local prefix='\* '
    local string=''
    for branch in $branches; do
        if [[ ${branch} == ${prefix}* ]]; then 
            string+=':['
            string+=${branch##$prefix}
            string+=']'
            break
        fi
    done
    echo $string
    unset IFS
}
```

The `${parameter##word}` expansion checks the beginning of `parameter` against the pattern `word`. If a match is found, the longest matching pattern is deleted.

To call this new function and have it `echo` that concatenated branch name into our prompt, we need to update the `PS1`:

```sh
# Begin appending information to PS1
export PS1=''

# Add cyan command-number: '\#'
PS1+=$'\e[1;36m\#.\e[0m '

# Add bold username: '\u'
PS1+=$'\e[1m\u\e[0m'

# Add bold-blue hostname: '\h'
PS1+=$'\e[1;34m@\h\e[0m in '

# Add red working directory: '\w'
PS1+=$'\e[1;31m\w\e[0m'

# Add git information
PS1+='$(git_branch) '

# Add green time: '\A'
PS1+=$'at \e[1;32m\A\e[0m'

# Add dollar-sign `$`
PS1+=$'\n$ '
```

This is good, and it works, but it has no color. When I was using oh-my-zsh, I had the branches name change colors based on the state of the repository:

- **Green branch**: Nothing to commit, all files clean.
- **Orange branch**: Files are in the staging area.
- **Yellow branch**: Local repo is ahead of remote.
- **Red branch**: Files has been changed, nothing in the staging area.

In order to replicate this functionality, we need a new function `git_dirty()`:

```sh
function git_dirty {
    local status=$(git status 2> /dev/null)
    local push='Your branch is ahead'
    local dirty='added to commit'
    local commit='Changes to be committed'
    
    # First check if the repo is dirty.
    if [[ $status =~ ${dirty} ]]; then echo $'\e[1;31m';
        
    # Next check if changes have been staged.
    elif [[ $status =~ ${commit} ]]; then echo $'\e[1;36m';
    
    # Check if its ahead of remote.
    elif [[ $status =~ ${push} ]]; then echo $'\e[1;33m';
    
    # Default to clean.
    else echo $'\e[1;32m';
    fi
}
```

`git_dirty()` parses the commit message, searching for one of the three canned substrings it knows to look for. Depending on which one it finds, it outputs an ANSI color code. This function could easily be called from `git_branch()`:

```sh
function git_branch {
    IFS=$'\n'
    local branches=$(git branch --no-color 2> /dev/null)
    local prefix='\* '
    local string=''
    for branch in $branches; do
        if [[ ${branch} == ${prefix}* ]]; then 
            string+=':['
            string+=$(git_dirty)
            string+=${branch##$prefix}
            string+=$'\e[0m'
            string+=']'
            break
        fi
    done
    echo $string
    unset IFS
}
```

This time `git_branch()` calls `git_dirty()` to get the color of the branch name. After it writes the branch name to the string, it resets the formatting and outputs the result. Because we're calling `git_dirty()` from `git_branch()` we don't have to update our prompt.

The final prompt:

<pre><span style="font-weight: 700; color:#d08770;">25. </span><span style="font-weight: 700;">pat</span>@<span style="font-weight: 700; color:#8fa1b3;">patmac</span> in <span style="font-weight: 700; color:#bf616a;">~/Code/dotfiles</span>:[<span style="color:#a3be8c;">master</span>] at <span style="font-weight: 700; color:#a3be8c;">17:11</span>
$</pre>

This is pretty great. Now I have the majority of my zsh functionality back, and it's all in a single bash startup file. The only major differences now are the in-buffer command highlighting [^highlight] and the completions &mdash; bash does allow you to write your own programmable completions.

All of my bash startup files &mdash; along with a slew of other dotfiles I use &mdash; are in [my dotfiles repository on Github][dotfiles].

---

[^zsh-alts]: There are oh-my-zsh alternatives, namely [Prezto][prezto]. It was actually a fork of oh-my-zsh that was later rewritten to focus on performance. But if I am going to *really* buy-in on zsh, I should either hunker down and learn the configurations or stick with oh-my-zsh.

[^fish]: Anyone who has ever tried to use the [fish shell][fish] can attest to the importance of your daily shell playing-nice with bash. This *is* a deal breaker.

[^trees]: Currently bash 4.4 has no dead tree version. So your stuck with reading the release notes for the latest features, like an animal.

[^c-code]: Commonly referred to as just "C code".

[^bsd]: Apple uses the BSD utilities, so any BSD system should operate the same.

[^ls]: All of this is contingent on your terminal being able to display colors, which is the default on most terminals today.

[^highlight]: This is pretty much impossible to do with bash. Zsh has [its own library][zshline] for command line buffers that enables this feature.

[bash]: http://tiswww.case.edu/php/chet/bash/bashtop.html
[zsh]: http://zsh.sourceforge.net
[omz]: http://ohmyz.sh
[zsh-slow]: https://kev.inburke.com/kevin/profiling-zsh-startup-time/
[zsh-slow2]: http://blog.patshead.com/2011/04/improve-your-oh-my-zsh-startup-time-maybe.html
[zsh-slow3]: http://superuser.com/questions/236953/zsh-starts-incredibly-slowly
[prezto]: https://github.com/sorin-ionescu/prezto
[fish]: http://fishshell.com
[windows]: https://msdn.microsoft.com/en-us/commandline/wsl/about
[license]: http://apple.stackexchange.com/questions/208312/why-does-apple-ship-bash-3-2
[brew]: http://brew.sh
[chet]: http://tiswww.case.edu/php/chet/
[bash-ref-html]: http://tiswww.case.edu/php/chet/bash/bashref.html
[bash-ref-pdf]: https://www.gnu.org/software/bash/manual/bash.pdf
[bash-ref-trees]: https://www.amazon.com/Bash-Reference-Manual-Chet-Ramey/dp/988838127X/ref=sr_1_1?s=books&ie=UTF8&qid=1474610205&sr=1-1
[ls-colors]: http://geoff.greer.fm/lscolors/
[ansi]: https://en.wikipedia.org/wiki/ANSI_escape_code
[git-prompt]: https://github.com/magicmonty/bash-git-prompt/blob/master/prompt-colors.sh
[ascii]: https://en.wikipedia.org/wiki/Escape_character#ASCII_escape_character
[ansi-colors]: https://en.wikipedia.org/wiki/ANSI_escape_code#Colors
[git]: https://git-scm.com
[git-shell-script]: https://github.com/git/git/blob/master/contrib/completion/git-prompt.sh
[zshline]: http://zsh.sourceforge.net/Doc/Release/Zsh-Line-Editor.html#Character-Highlighting
[dotfiles]: https://github.com/patrickrgaffney/dotfiles