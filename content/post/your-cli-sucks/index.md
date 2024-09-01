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

### they have a low chance of working out of the box

I often see CLI tools that are very hard to use unless some arbitrary set of conditions is met and that set of conditions is not adequately communicated to the user.

### they fail silently or they fail with indecipherable error messages

handle your errors and add context to your errors when you bubble them up to the user! There is nothing worse than getting a silent failure and sitting there like:
"Uhhh, did something happen? what did it do?"
or having to dig through 500 lines of plain white garbage text to find out where things went wrong, only to spend 2 hours to figure out that some obscure flag was
required for some arbitrary reason.

### not providing some sort of visual feedback when something is happening in the background.

use a spinner or a progress bar for long running tasks or at the very least log to stderr what is happening
bonus points if you offer different levels of logging and info using a flag like `-v` or `-vv` these things
are incredibly simple to do in most langauges, manually or with a library.

### endless walls of plain white text. (looking at you `apt`)

it is `$CURRENT-YEAR`, use some effing _ansi colors_... There are a large amount of amazing libraries that will
completely handle this for you and all the things you need to watch out for like:

- respecting the $NO_COLOR environment variable standard for environments that don't allow for color or user preference

- detecting if the CLI tool's stdout is connected to a terminal or a pipe and context switching based on that. You don't want to pipe ansi sequences into other tools
  color is important in the terminal. Color is the universally understood language. You can clearly indicate that things are going well with green, or log warnings with yellow,
  or print errors with red. Error messages should be clear, what the user can/should do to remedy it should be clear, and an error should be a visible red and easily identifiable at a glance among walls of text

But honestly, you do not even need a library. Ansi codes are not that hard!

```go
// reset color
const CLEAR = "\x1b[0m"

// 8 colors goes from \x1b[31m to \x1b[39m
const RED = "\x1b[31m"
// ... etc...

// TRUE COLOR
foreground = "\x1b[38;2;" + R + ";" + G + ";" + B + ";"
background = "\x1b[48;2;" + R + ";" + G + ";" + B + ";"

```

and thats basically it! You can look up the codes for italic and bold, and bonus points if you allow a `--no-color` option and respect the `$NO_COLOR` environment variable
this reallyyyyyy ups the quality of your CLI tool, and I don't think I've used a terminal that doesn't support the basic ansi colors (and most do true color)
bonus bonus points if you check the `$TERMINFO` entry for truecolor support (there are also other ways to do this) before slinging in a bunch of ansi codes

### not respecting the users system (warning: this is a long one, but important)

these tools often don't respect XDG defaults (among all systems) when creating files. I even see far _too many_ enterprise applications who do this as well.
Android Studio is a great example. These apps will litter your `$HOME` directory with config files, cache files, and system files with reckless abandon.
So much so that people have developed tools to remedy this problem ![xdg-ninja](https://github.com/b3nj5m1n/xdg-ninja) and it has 2.7K stars to boot. Just go peep
at that changelog to see the worst offenders. (hint many of them are large enterprise projects from Google, Microsoft, Azure, etc...) It's also unacceptable
that this is also the case for the `.bashrc`, granted it is for legacy reasons, but it has been long enough and it needs to be changed.

The simple solution is to allow the user to specify a location for config/cache/other files using the mysterious and elusive `$environment_variable`

for example:

```sh
BASH_DIR=~/.config/bash
ANDROID_DATA_DIR=~/.cache/android-studio
```

let me introduce you to the ![XDG basedir specification](https://specifications.freedesktop.org/basedir-spec/latest/index.html)
It allows the system to define sensible defaults for specific directories that you would expect to exist... Using environment variables allows
the user to override these locations so that they can easily define where files should go. There are a lot of benefits to this.

```sh
# the most common
XDG_DATA_HOME="$HOME/.local/share"
XDG_CONFIG_HOME="$HOME/.config"
XDG_CACHE_HOME="$HOME/.cache"
XDG_STATE_HOME="$HOME/.local/state"

# the typical xdg-user-dirs
XDG_DESKTOP_DIR="$HOME/Desktop"
XDG_DOWNLOAD_DIR="$HOME/Downloads"
XDG_TEMPLATES_DIR="$HOME/Templates"
XDG_PUBLICSHARE_DIR="$HOME/Public"
XDG_DOCUMENTS_DIR="$HOME/Documents"
XDG_MUSIC_DIR="$HOME/Music"
XDG_PICTURES_DIR="$HOME/Pictures"
XDG_VIDEOS_DIR="$HOME/Videos"
```

In the case that these variables aren't defined, you can fall back to:

```sh
# the most common
"$HOME/.local/share"
"$HOME/.config"
"$HOME/.cache"
"$HOME/.local/state"
# the typical xdg-user-dirs
"$HOME/Desktop"
"$HOME/Downloads"
"$HOME/Templates"
"$HOME/Public"
"$HOME/Documents"
"$HOME/Music"
"$HOME/Pictures"
"$HOME/Videos"
```

It's genuinely pretty simple. Do you want to guess where you can fallback after that?
Yes! After checking these places, or creating files in these places unsuccessfully, you can use `$HOME`...
Thats IF the user hasn't passed a CLI specific env var that you have provided that configures where files are kept OR
a cli flag that does the same thing.

So config files should attempted to be read and created in this order:

- A user defined location via environment variables, or directly through flags (similar precedence)

```sh
# your env prefix should be specific to the tool so it doesn't clash with other env-vars. ex: $MY_CLI_
MY_CLI_CONFIG="$HOME/.config/my-cli/config.conf"
MY_CLI_CACHE_DIR="$HOME/.cache/my-cli"
./my-cli --config "$HOME/.config.conf"
```

and in my opinion, the flag should trump the environment variable since we are explicitly passing that to the tool last.

...
and then we would check for files in this following order:

> `$XDG_CONFIG_HOME` -> `$HOME/.config` -> or `$HOME/`

In some cases `/etc/my-cli` is acceptable, especially if the CLI tool is expected to be run as root or shared across many users.

But the main point is that there should be an established precedence for that is respected when looking for or creating config/cache files... with the ability
for the user to specify their location manually.

The biggest thing is to allow this to be dynamic, let the user decide where things should go. This requires you to design a CLI tool with this in mind. In many
ways this is easier than hard-coding a path and hard failing when the file is not in that location. A CLI tool should be a lot more resistant to failure than that,
because not having a config file is the expected default state when you use a tool for the first time.

You can remedy this by either offering sensible defaults when a config file doesn't exist OR providing a `--dump-config` option
where you print a config to `stdout`

It is extremely annoying when programs create cache, config, or data files in `$HOME`. It is disrespectful of the users personal space especially when they don't allow the user to
specify another location. This location should be dynamic.

Here is a good example of the Golang stdlib that finds the users `$HOME` dir for every platform:
(you can substitute this for `$XDG_CONFIG_HOME` or `$XDG_CACHE_HOME`)

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

If you are unsure of what to do, I genuinely recommend just ganking the code from the Golang stdlib. They have done all of the hard work here and you can just
copy and translate it to your language. Then as a developer, I do something like this:

```go
configLocations := []string{
	"$XDG_CONFIG_HOME/my-cli",
	"$HOME/.config/my-cli",
	"$HOME/.config/my-cli",
}

type MyConfig struct {
	configFilePath string
}

func (m *MyConfig) FindConfig(paths []string) (string, error) {
	for _, path := range paths {
		realpath := os.ExpandEnv(path)
		// you can even run filepath.Abs(realpath) to follow symlinks and stuff
		_, err := os.Stat(realpath)
		if err == nil {
			return realpath, nil // stat checks if it exists, so no error == we found the file
		}
	}
	return "", errors.New("config file not found bruh U_U")
}

```

or something along these lines. You will just have to offer the best possible defaults and trust that you made a program that is fault tolerant enough
so that it is pretty hard to get wrong.

Sometimes I get the vibe that people see Linux and such as just a "container", "server" or "WSL" thing and therefore littering garbage wherever you please is perfectly fine.
This isn't true and its a dick move, when it costs you basically nothing to allow people to override a path with an environment variable at the very least.
I certainly expect more from Azure, Google, Android, etc... or the other 2,000 teams on the xdg-ninja list. Its unacceptable.
