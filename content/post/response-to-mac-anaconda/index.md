+++
title = 'Response to Mac Anaconda'
date = 2024-09-01T11:11:18-08:00
author = "sweetbbak" 
+++

A response to a post on HackerNews that irked me regarding Anaconda and teaching students

<!--more-->

This article was on HackerNews recently [paulromer.net/escaping-from-anaconda/](https://paulromer.net/escaping-from-anaconda/) and I really couldn't believe what I was reading...

A quote from the article:

> If you teach a course that requires Python, you’ve encountered students on macOS who are trapped by Anaconda.
> Typically, they were told to install Anaconda in a first course on Python or data analytics.
> After that course is over, they discover that they can’t get an official version of Python to run.
> Many of these students have not used the command line. Many do not understand the difference between a text editor and a word processor.
> If they ask ChatGPT for help, it responds with instructions that they do not understand and that wouldn’t work anyway. In my experience, most give up.

Then... shouldn't this be your job? To teach the students how to use the tools that are necessary to developing and learning literally any programming language?

> This post gives them a simple way to recover the ability to run an official version of Python.

This post goes on to go through an overly complicated process of hitting the exact MacOS specific shortcuts, and menu navigation... all too literally just
move the default `.zshrc` file so that it isn't loaded, ALL because Anaconda adds a line to the shell rc file when it is installed.
which I agree that this is extremely annoying, these tools do not respect a users machine and act like they are only running on WSL, servers or containers.
A LOT of enterprise level tools conform to bad practice. Just refer to my rant on [why your cli sucks]({{< ref "/post/your-cli-sucks/index.md#not respecting the users system (warning: this is a long one, but important)" >}} "respecting the users system") in the "respecting users systems" section
where a describe a lot of these bad practices alongside the laundry list of projects that violate best practice, and the `xdg-ninja` tool that is dedicated to fixing this.

This is exactly why you should learn to use your tools properly, learn just a tiny tiny tiny bit of posix shell (which works for ALL posix shells sh, tcsh, bash, zsh etc...) so that you can understand what the heck is going on.
Like seriously, this is day one stuff. Your shell loads a configuration file and you can set programs to run on start, or configure shell options, set aliases etc...
As a teacher, you have somewhat of a responsibility to teach people the proper method, and not dump shortcuts at their feet that are fragile and won't hold up to the
test of time.

It's especially baffling, when the solution is as simple as commenting out the line that Anaconda adds to your default shell rc file.

## Solutions

Anaconda provides a silent mode while installing (and an after install config option) where you can speicfy where conda is installed
![anaconda silent mode](https://docs.anaconda.com/anaconda/install/silent-mode/)

Here is _their_ example:

```fish
curl https://repo.anaconda.com/miniconda/Miniconda3-latest-MacOSX-x86_64.sh -o ~/miniconda.sh
# OR for Linux
wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh -O ~/miniconda.sh
bash ~/miniconda.sh -b -p $HOME/.cache/miniconda
```

where `-b` is described as "Batch mode" with no PATH modifications to ~/.bashrc. Assumes that you agree to the license agreement. Does not edit the .bashrc or .bash_profile files.
and where `-p` is the install `$PREFIX`.

to activate in the `$CURRENT` shell run:

```fish
# ie shell.bash shell.zsh
eval "$(/$HOME/miniconda/bin/conda shell.YOUR_SHELL_NAME hook)"
# AND
conda init
# and if your prefer the base env to NOT be auto activated
conda config --set auto_activate_base false
```

consider using `Nix` on Mac or Linux for reproducible builds (though Python is a nightmare to configure no matter what tool you use) or `daytona` (which I've heard good things about)
or default `venv` or one of the many like it... or a `Containerfile` (agnostic podman/docker file).

Configuring Python environments is extremely painful, idk if there is a way around that. I will often have every given method fail whether that be `Docker` or `conda` or `venv` with `pip`
I'm not exactly sure how you could make it better, but I do think the state of things regarding Python is genuinely insane, especially given how popular it has gotten
for AI and LLMs... and all the billions of dollars floating around in the ecosystem.
