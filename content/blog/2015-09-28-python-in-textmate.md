+++
title = "Python in TextMate 2"
description = "Get your Python virtual environments working in TextMate 2."
slug = "python-in-textmate"
date = 2015-09-28
draft = false
template = "post.html"

[taxonomies]
  tags = ["python", "textmate", "virtualenv", "editors"]
+++

The true dominance of [TextMate][mate] lies in its [bundles][mate-bundles].  No matter what language you prefer to spend your time writing, chances are, there exists a bundle that is feature-packed with the idioms, syntax, and completions of that language.

TextMate ships with a [Python bundle][python-bundle], and it contains a multitude of *already* useful commands. [^commands]  But, alas, it's "Run Script" command, invoked with `⌘R`, refuses to cooperate with me.  TextMate chooses to use the default installation of Python that [ships with OS X][osx-py].  To get specific, this is version 2.7.10 on El Capitan.  This is a problem because I have decided to [forgo living in the past][python-2-3], and otherwise embrace the future that is Python 3.

## #!/use/the/shebang

TextMate will parse the first line of your Python script for a [shebang][shebang] sequence.  If found, it will attempt to call that interpreter when you run your script.  I have installed Python 3 using [homebrew][brew], so the binary is in my `/usr/local/bin` folder.  Lets set up a simple script and test this:

```py
#!/usr/local/bin/python3
print('hello, hypepat!')
```

!["hello, hypepat!"][hello]

Excellent, it worked.  It did everything I wanted: 

- use the `python3` interpreter
- don't leave TextMate to find the terminal
- print to a new window [^window]

But, this is not very pragmatic.  We shouldn't have to rely on the shebang line to find the interpreter.  Surely TextMate provides a way to find the interpreter I want, on an application-wide basis.

## Rage Against the Defaults

First some background: TextMate uses [environment variables][mate-envs] to provide context to its various scripts and commands, including those in bundles.  [**Dynamic variables**][mate-d-envs] are defined by TextMate and provide the current configuration of the application. [^dynamic-envs]  [**Static variables**][mate-s-envs] are defined by the user in Preferences → Variables and give application-wide support to commands and snippets.  Finally, [**Project dependent variables**][mate-pd-envs] are defined by the user in their project configuration file, and exist only for snippets and commands executed in the context of that project.

Now, if we were to do some digging into the internals of the Python bundle, [^py-bundle] we will find a Ruby script that is called every time we run our code.  Line #13 of that script actually calls the Python interpreter to run our script, and the code looks alot like the following:

```rb
TextMate::Executor.run(ENV["TM_PYTHON"] || "python", "-u", ENV["TM_FILEPATH"] ...
```

It appears there is a static variable, `TM_PYTHON`, that is either set to the OS X system interpreter, or isn't set at all.  We could update the `"python"` after the logical-or to `"python3"`, but this single line is going to be executed for every script we run in the future.  And to be honest, there is more Python 2 code than Python 3 code in the world, so that seems like a rash thing to do.

Instead, lets assign a value to `TM_PYTHON`.  Static variables are assigned in Preferences → Variables.  Hit the `+` and enter `TM_PYTHON` for the "Variable Name".  Now we can enter the absolute path to our `python3` interpreter, as we did above with the shebang line.  Go ahead an enter `/usr/local/bin/python3` for the "Value" of the variable.  Rerun the script, and marvel at your command of a very powerful editor.  

Now, go back and disable (uncheck) the static variable `TM_PYTHON` in Preferences → Variables, and lets try something perhaps a little more elegant.

## A Product of the Environment

In most cases manually changing the value of `TM_PYTHON` is not the most elegant solution, as we are limiting ourselves to the use of only one interpreter.  And considering the *vast* majority of Python development is done in [virtual environments][venv], this is not an option.

Virtual environments work by **prefixing** your `$PATH` shell variable with the location of a locally installed Python interpreter and library. [^locally]  This way, when you run your scripts, they call the `python` that was installed through [`virtualenv`][venv].  This allows the developer to create specific environments with specific dependencies.  Traditionally, the developer would use `/usr/bin/env` instead of an absolute path to specify the first interpreter found along the `$PATH`.

So, lets backtrack a bit and re-examine that shebang line.  We can change it from `/usr/local/bin/python3`, the absolute path, to `/usr/bin/env python3`, the environment variable.

```py
#!/usr/bin/env python3
print('hello, hypepat!')
```

![TextMate path error][tm-path-error]

Damn it.  Although our `$PATH` variable is set, TextMate [does not parse our shell scripts][tm-path].  In other words, TextMate doesn't have any idea what our `$PATH` variable is set to, because it doesn't share environment variables with the shell.  It exports the dynamic, static, and project-dependent variables when a command is issued from within the editor, as needed. [^error]

Fortunately, we can assign our `$PATH` variable to be recognized in TextMate.  Venture back to Preferences → Variables and add a variable with the name `$PATH` and the value `$PATH:/usr/local/bin`. [^brew]  Now, running our script again should be successful. [^path]

Now TextMate is dynamically pointing at the first Python 3 interpreter in our `$PATH`.  This is okay, but it isn't a great solution.  For example, if we were to edit our `$PATH`, perhaps by entering a `virtualenv`, TextMate would have no idea.  Thus, we have more work to do.

## Configuring the Configuration

Exit: frustration, enter: [TextMate preferences][tm-prefs]!  TextMate allows us to create project-specific settings under the umbrella of a `.tm_properties` file.  You may remember "project dependent variables" from eariler.  These are just what we need to make our solution virtual-environment-proof.

First, a few basics on these preferences files.  To start, all we need to do is dump a `.tm_properties` file into the root directory of our project.  In fact, any folder that has a `.tm_properties` overrides any *parent* folder with a properties file.  In other words, you can nest these files in your projects to specify settings on a folder-level.  And considering TextMate 2 treats every folder as a project, the possibilities are really endless.

After opening that file in TextMate, you will see that it has enabled the "TextMate Settings" language syntax.  These settings work just as basic assignments, but their power is in their override-ability.  Any *global* settings you have set up in Preferences → Variables can be overridden in this file, as your specific project calls for it.

Lets see a simple example:

<div class="highlight">
<pre>
<span class="k">TM_PYTHON </span><span class="o">= </span><span class="s">"</span><span class="se">${CWD}</span><span class="s">/venv/bin/python"</span>
<span class="k">excludeDirectoriesInBrowser </span><span class="o">= </span><span class="s">"{__pycache__,include,lib,bin}"</span>
<span class="k">includeFilesInBrowser </span><span class="o">= </span><span class="s">"{.gitignore}"</span>

<span class="o">[ </span><span class="sx">source.python</span><span class="o"> ]</span>
<span class="k">softTabs </span><span class="o">= </span><span class="bp">true</span>
<span class="k">tabSize  </span><span class="o">= </span><span class="m">4</span>
</pre>
</div>


The first setting overrides the global `TM_PYTHON` variable we assigned earlier to use an interpreter that is located in a subfolder of our current working directory, or `${CWD}`.  If you are using `virtualenv`, the "venv" folder contains your python interpreter along with all its libraries.  If you name your virtual environments, then this folder will bear that same name.

The second and third lines tell TextMates file browser to exclude and include specific files for this project.  [CPython][cpython] often utilizes cache files to optimize code, and I am choosing to have these files excluded from cluttering up my file brower.  Also, I would like to see my ".gitignore" file in the browser, but because it is a [hidden file][hfiles], it is excluded by default.  You can see just how powerful these `.tm_properties` are with a few settings.

Finally, Python takes a very strict stance on whitespace.  No matter your feelings on this, we need to make sure that indentation is four spaces.  We can set global settings in this file for all files of the project that have a specific syntax, using the `source.«language»` modifier.


There are [tons of possible settings][tm-settings], and if you're short on ideas you can checkout the [default settings][tm-defaults].

Using these settings files we can pin down the exact location of our python interpreter, without listing an absolute path in the shebang line.  This also keeps us from manually changing our `TM_PYTHON` static variable each time we switch virtual environments.  The creation of these files can even be strung together with the `virtualenv` command to create a virtual environment and specify the settings for the editor all in one command.


[^commands]: Which, after having a look at some of the [competition][choc], is TextMate's biggest draw for me to continue using it even after its... [retirement][death-of-TM]? Hell, I'm not even sure what to call it.

[^window]: The default settings for TextMate show the output on the right of the window in a drawer, but this resizes the editor, and we can't have that.  Better to have it in a pop-up that we can quickly `⌘W` and get back to our code.  Plus the pop-up window works great in full-screen mode.  Change this setting in Preferences → Projects.

[^dynamic-envs]: AKA: they are *implictly* changed by the user, just by altering the state of the application.

[^py-bundle]: You can edit bundles by going to Bundles → Edit Bundles, or `^⌥⌘B`.

[^locally]: *Locally* meaning "usually inside your current working directory", or otherwise specified when you created the virtual environment.  Unless of course, you're using [`virtualenvwrapper`][venv-wrap], in which case "usually inside a hidden folder in your home directory."

[^error]: If you didn't get an error here, that's fine.  It just means you didn't uncheck your `TM_PYTHON` variable earlier.  You can continue along just the same.

[^brew]: I am keeping `/usr/local/bin` in my `$PATH` because I use homebrew to install programs, and that's where it installs them.  If you are using [MacPorts][macports] (then switch to homebrew...) then you will use `$PATH:/opt/local/bin`.

[^path]: If you are getting a `$PATH` related error still, you probably need to update your path variable.  Depending on which login shell you prefer, you can probably update the `$PATH` in your [startup files][startup].

[mate]: http://macromates.com
[mate-bundles]: http://manual.macromates.com/en/bundles#bundles
[python-bundle]: https://github.com/textmate/python.tmbundle
[choc]: https://chocolatapp.com
[death-of-TM]: http://blog.macromates.com/2012/textmate-2-at-github/
[osx-py]: http://docs.python-guide.org/en/latest/starting/install/osx/
[python-2-3]: https://wiki.python.org/moin/Python2orPython3
[shebang]: http://en.wikipedia.org/wiki/Shebang_%28Unix%29
[brew]: http://brew.sh
[mate-envs]: http://manual.macromates.com/en/environment_variables
[mate-d-envs]: http://manual.macromates.com/en/environment_variables#dynamic_variables
[mate-s-envs]: http://manual.macromates.com/en/environment_variables#static_variables
[mate-pd-envs]: http://manual.macromates.com/en/environment_variables#project_dependent_variables
[venv]: http://virtualenv.readthedocs.org/en/latest/
[tm-path]: http://blog.macromates.com/2014/defining-a-path/
[startup]: http://hyperpolyglot.org/unix-shells#startup-file
[macports]: http://www.macports.org
[tm-prefs]: http://blog.macromates.com/2011/git-style-configuration/
[cpython]: https://en.wikipedia.org/wiki/CPython
[hfiles]: https://en.wikipedia.org/wiki/Hidden_file_and_hidden_directory#Mac_OS_X
[tm-settings]: http://wiki.macromates.com/Reference/Settings
[tm-defaults]: http://wiki.macromates.com/Reference/FolderSpecificSettings
[venv-wrap]: http://virtualenvwrapper.readthedocs.org/en/latest/

[hello]: /assets/hello_hypepat.png
[tm-path-error]: /assets/mate_path_error.png