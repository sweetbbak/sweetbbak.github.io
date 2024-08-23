+++
title = 'Development Containers'
date = 2024-08-02T11:48:30-08:00
+++

How to use Podman, Docker or Lilipod containers to do simple and seamless development on Linux without cluttering up your system.

<!--more-->

I have run into a problem many times in my time developing on Linux where I would be trying to compile some arcane software, or attempting to
try and use some obscure C library that requires a handful of dependencies... so I go on an installing spree, installing random libs
in an attempt to resolve LSP and compiler issues. This quickly becomes messy and a lot of these things end up staying on my system forever.

On the other hand I also use NixOS where I can define these dependencies using nix... which is definitely a step up, it comes with its own set of
issues. Mainly I run into issues with tools like clang not being able to find libs and then is unable to provide any useful LSP definitions or completions. In many cases
the software will still compile (and in many cases, not) but it can be equally frustrating. `pango` and `cairo` are a great example of this.

So what is the solution here? What can I do?

```bash
#!/usr/bin/env bash
set -euo pipefail
user_uid=$UID

podman run \
    --rm \
    --security-opt label=disable \
    -e XDG_RUNTIME_DIR=/tmp \
    -e "WAYLAND_DISPLAY=$WAYLAND_DISPLAY" \
    -v "$XDG_RUNTIME_DIR/$WAYLAND_DISPLAY:/tmp/$WAYLAND_DISPLAY:ro" \
    -it \
    "$@"
```

If you are using X11, the equivalent will look something like this:

```bash
-v $XAUTHORITY:$XAUTHORITY:ro \
-v /tmp/.X11-unix:/tmp/.X11-unix:ro
```

Here's what it does:
This option sets the security label for SELinux and seccomp.

- `--security-opt="label=level:LEVEL"`: Set the label level for the container.

  We are going to set the label to "disable" to disable this feature. This makes it makes it easier to run GUI applications in the container.

- `--security-opt="label=disable"`: Disable the SELinux configuration for the container.

Here we set an environment variable defining the `$XDG_RUNTIME_DIR` as `/tmp`

- `-e XDG_RUNTIME_DIR=/tmp`

and then we mount the Wayland display socket at tmp (which the container will find using the `$XDG_RUNTIME_DIR` env var) which will allow the Wayland or X11
server to talk to communicate with the applications and programs inside the container.

`--rm` removes the container after exit, and `-it` is short for `--interactive` and `--tty` indicating that we want to run an interactive terminal session and not
a detachted background process. You can remove `--rm` if you wish for the container to persist past exit.

# Creating a custom container that has everything that we need...

Lets start by creating a containerfile that tells podman or docker how to build our container image.
You can use whatever Linux distro and dev environment that is comfortable to you. I am going to use `arch btw` and my basic development tools.

lets create a filed called `Containerfile` and edit the contents to the following:

```Dockerfile
# pull the image, other examples are debian:latest, ubuntu:21.04 etc...
FROM archlinux:latest
# Update locale
RUN echo "en_US.UTF-8 UTF-8" >> /etc/locale.gen && locale-gen

# Update package repository
RUN pacman -Syyu --noconfirm

# Install dev tools
RUN pacman -S --noconfirm \
    git base-devel \
    helix neovim zsh fish wget curl \
    eza zoxide tmux fd bat ripgrep \
    go rust starship fzf \
    jq rsync man-db \
    unzip glibc python nix

# add a user
RUN useradd -m -G wheel sweet

# delete the password so we won't need to use it
RUN passwd -d sweet
RUN printf 'sweet ALL=(ALL) ALL\n' | tee -a /etc/sudoers
RUN chsh --shell /bin/zsh sweet

# install paru for the AUR
RUN su - sweet -c sh -c 'git clone https://aur.archlinux.org/paru.git && cd paru && makepkg -s --noconfirm && sudo pacman --noconfirm -U *.pkg.tar.zst'

# install paruz fzf AUR package installer
RUN curl -fsSl https://raw.githubusercontent.com/joehillen/paruz/master/paruz > /bin/paruz
RUN chmod +x /bin/paruz

# Use the host install of podman as podman inside the container
RUN ln -s `which distrobox-host-exec` /usr/local/bin/podman

# zsh prompt example
# RUN echo 'setopt PROMPT_SUBST' | tee -a /home/sweet/.zshrc
# RUN echo 'setopt autocd' | tee -a /home/sweet/.zshrc
# RUN echo "export PROMPT='%F{green}%*%f %F{blue}%~%f %F{red}${vcs_info_msg_0_}%f 📦 arch-dev $ '" | tee -a /home/sweet/.zshrc
# RUN echo 'eval $(starship init zsh)'  | tee -a /home/sweet/.zshenv
```

then when we run our little shell script we will want to specify some extra things on the command line:

```bash
-v /home/sweet/.config/starship/:/home/sweet/.config/starship \
-v /home/sweet/.config/zsh:/home/sweet/.config/zsh:ro \
-v /home/sweet/.config/nvim:/home/sweet/.config/nvim:ro \
-v /home/sweet/.zshenv:/home/sweet/.zshenv:ro \
```

here we mount our useful dotfiles as `read-only` into the container (`-v` is short for `--volume` which is essentially a mount)
we could also add this to our script so that we don't need to add it each time.

then finally we run our container!

```bash
pod.sh -v $PWD:$PWD archdev:latest su - $USER -c zsh
# OR
pod.sh --userns keep-id --user 1000 -v $PWD:$PWD localhost/archdev:latest "zsh"
```

<hacker voice> ...aaaaand I'm in.

be careful of mounting your home directory in a container, you can obviously still delete things... so do NOT go running `rm -rf /*` thinking you are safe

Now, you can develop and install trash libs to your hearts content :3 :heart:

try something like:

```bash
sudo pacman -Syyu wofi fd --noconfirm
fd . /bin | wofi --show dmenu
```

and see that it opens on the host machine! If you get really handy with it, you can even run Window Managers in a container.
(you could also just use Distrobox lmao)
