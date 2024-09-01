+++
title = 'Why your CLI programs suck!'
author = 'sweetbbak'
date = 2024-07-27T11:05:00-08:00
draft = false
+++

How to create CLI programs that don't suck

<!--more-->

# Why your CLI programs suck!

One of my favorite things to do is to paruz through github searching for interesting software and CLI tools. This can be a great method of learning
how other people solve problems. I have about 3,500 stars or so, purely from just doing this. It's a great way to see how to do things correctly, and how to do
things incorrectly.

I can't tell you how many times that I come across a unique and interesting tool, that frankly, has an awful user interface... and I find that to be somewhat tragic.

Here are some very common mistakes that I see people make:

- they have a low chance of working out of the box
  I often see CLI tools that are very hard to use unless some arbitrary set of conditions is met
  and that set of conditions is not adequately communicated to the user.

- they fail silently or they fail with indecipherable error messages
  handle your errors and add context to your errors when you bubble them up to the user!

- not providing some sort of visual feedback when something is happening in the background.
  use a spinner or a progress bar for long running tasks or at the very least log to stderr what is happening
  bonus points if you offer different levels of logging and info using a flag like `-v` or `-vv` these things
  are incredibly simple to do in most langauges, manually or with a library.

- endless walls of plain white text. (looking at you `apt`)
  it is $CURRENT_YEAR, use some effing _ansi colors_... There are a large amount of amazing libraries that will
  completely handle this for you and all the things you need to watch out for like:

  - respecting the $NO_COLOR environment variable standard for environments that don't allow for color or user preference

  - detecting if the CLI tool's stdout is connected to a terminal or a pipe and context switching based on that. You don't want to pipe ansi sequences into other tools
    color is important in the terminal. Color is the universally understood language. You can clearly indicate that things are going well with green, or log warnings with yellow,
    or print errors with red. Error messages should be clear, what the user can/should do to remedy it should be clear, and an error should be a visible red and easily identifiable at a glance among walls of text

- they don't respect XDG defaults (among all systems) when creating files.

Config files should attempted to be read and created in this order:

> A user defined location via environment variables, or directly through flags ie:

```sh
# your env prefix should be specific to the tool so it doesn't clash with other env-vars. ex: $MY_CLI_
MY_CLI_CONFIG="$HOME/.config/my-cli/config.conf"
MY_CLI_CACHE_DIR="$HOME/.cache/my-cli"
./my-cli --config "$HOME/.config.conf"
```

and then in this following order:

> `$XDG_CONFIG_HOME` -> `$HOME/.config` -> or `$HOME`

In some cases `/etc/my-cli` is acceptable, especially if the CLI tool is expected to be run as root or shared across many users.

There should be an established precedence for which method is respected if multiple are provided. We can assume that if someone is using the `--config` flag
that this would be the config path the user wants to be respected even if others exist.

The biggest thing is to allow this to be dynamic, let the user decide where things should go. This requires you to design a CLI tool with this in mind. In many
ways this is easier than hard-coding a path and hard failing when the file is not in that location. A CLI tool should be a lot more resistant to failure than that
because not having a config file is the expected default state when you first use a tool. You can remedy this by either offering sensible defaults when a config file
doesn't exist OR providing a `--dump-config` option where you print a config to `stdout` specifically, or auto-creating a config.

It is extremely annoying when programs create cache, config, or data files in `$HOME`. It is disrespectful of the users personal space especially when they don't allow the user to
specify another location. This location should be dynamic.

Finding HOME in Go:

```go
// If the expected variable is not set in the environment, UserHomeDir
// returns either a platform-specific default value or a non-nil error.
func UserHomeDir() (string, error) {
	env, enverr := "HOME", "$HOME"
	switch runtime.GOOS {
	case "windows":
		env, enverr = "USERPROFILE", "%userprofile%"
	case "plan9":
		env, enverr = "home", "$home"
	}
	if v := Getenv(env); v != "" {
		return v, nil
	}
	// On some geese the home directory is not always defined.
	switch runtime.GOOS {
	case "android":
		return "/sdcard", nil
	case "ios":
		return "/", nil
	}
	return "", errors.New(enverr + " is not defined")
}
```

finding the default config dir:

- Windows
  `Getenv("AppData")`

- Mac and Ios

```go
dir = Getenv("HOME")
if dir == "" {
	return "", errors.New("$HOME is not defined")
}
dir += "/Library/Application Support"
```

- plan9 (home is lower case)
  `$home` + `/lib`

- linux
  `$XDG_CONFIG_HOME` OR `$HOME` + `/.config`

```go
func UserCacheDir() (string, error) {
	var dir string

	switch runtime.GOOS {
	case "windows":
		dir = Getenv("LocalAppData")
		if dir == "" {
			return "", errors.New("%LocalAppData% is not defined")
		}

	case "darwin", "ios":
		dir = Getenv("HOME")
		if dir == "" {
			return "", errors.New("$HOME is not defined")
		}
		dir += "/Library/Caches"

	case "plan9":
		dir = Getenv("home")
		if dir == "" {
			return "", errors.New("$home is not defined")
		}
		dir += "/lib/cache"

	default: // Unix
		dir = Getenv("XDG_CACHE_HOME")
		if dir == "" {
			dir = Getenv("HOME")
			if dir == "" {
				return "", errors.New("neither $XDG_CACHE_HOME nor $HOME are defined")
			}
			dir += "/.cache"
		}
	}

	return dir, nil
}
```
