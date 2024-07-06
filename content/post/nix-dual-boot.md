+++
author = "sweetbbak"
title = "dual booting nixOS and linux"
date = "2024-05-31"
description = "How to dual boot NixOS with Linux"
tags = [
    "nix",
    "markdown",
    "text",
]
+++

How to dual boot NixOS with any generic Linux installation.

<!--more-->

# How to dual-boot NixOS with Linux

This is a simple tutorial on how to dual boot NixOS alongside an existing Linux installation.
This tutorial will show you how to install NixOS into an existing boot partition, making it available
in the current bootloader configuration. I will also lightly touch on the mounting an existing home directory
and any drives you may have.

This is tested on Apple Mac books, Linux with systemd-boot and it should work the exact same with
Linux and Grub.

## Instructions

Put NixOS on a USB drive. I personally used Ventoy.
Boot into NixOS, close that installer and pull up a terminal! (I know, scary right? but don't worry)

lets get a good idea of what the disks situation looks like, run:

```fish {linenos=table,hl_lines=[1,"15-17"],linenostart=1}
lsblk -fa

```

This will output our disks alongside their UUID

(edited for brevity)

```fish
NAME        FSTYPE FSVER LABEL    UUID                                 FSAVAIL FSUSE% MOUNTPOINTS
sda
└─sda1      ext4   1.0            9fd4533e-aa5d-42d4-8f47-bdfccc870492    2.6T    21% /home/sweet/hdd
nvme1n1
├─nvme1n1p1 vfat   FAT32          D888-C228                             462.1M    54% /boot
├─nvme1n1p2 ext4   1.0   home     9c8d0fde-5902-4cdf-8c97-67a93b031f7e  366.6G    51% /home/sweet/eos
└─nvme1n1p3 ext4   1.0   EOS      3ba30b0b-d02a-4791-b3c1-53a4d922c978
nvme0n1
└─nvme0n1p1 ext4   1.0            12c1a8df-e095-42d9-aca3-7a2a8c124dd8    1.1T    36% /home/sweet/sdx
nvme2n1
```

At this point I have some familiarity with which drive is which, in my case
nvme2n1 is completely empty and unused. This is where I will be installing NixOS into.
You can use _any_ unused partition.

### Create a partition and format it

Now may be a good time to open up gparted that comes with the live disk (or your favorite partitioning tool)
and sort this out. Create an empty partition if you need to or use an existing empty partition.
Then format that partition in your desired disk format, I used ext4 since it is simple and standard.
Remember to take note of the partitions full disk name (for example, nvme1n1p1 or sdb1 for hard disks)

### Mount the root and boot partitions

Next, if you haven't opened that terminal yet, do so. Lets mount that partition at `/mnt`

```fish
sudo mkdir -p /mnt
sudo mount /dev/nvme2n1p1 /mnt
# remember to replace nvme2n1p1 with your disk partition
# sudo mount /dev/{disk_part} /mnt
```

Next, we will create the `boot` directory inside the mounted partition:

```fish
sudo mkdir -p /mnt/boot
```

Now we want to take our existing EFI partition and mount it at `/mnt/boot`
My existing EFI partition is on disk `nvme1n1` partition `nvme1n1p1` it is typically labeled
as `FAT` or `VFAT` and may or may not have a label indicating that it is an EFI or BOOT partition.

```fish
sudo mount /dev/nvme1n1p1 /mnt/boot
```

### Optionally mount your home directory

---

** OPTIONAL **
optionally, you can also mount your existing home partition at `/mnt/home` if you want to.
You can also do this step at a later time so if you are having trouble deciding there is no
harm in waiting either.

```fish
sudo mkdir -p /mnt/home
sudo mount /dev/existing_home_partition /mnt/home
```

---

## Generate the NixOS config

next we are going to generate our Nix config that will direct the nix-install
Run:

```fish
nixos-generate-config --root /mnt
```

This will generate a config file at `/mnt/etc/nixos/configuration.nix`

In the generated config file add:

```nix
nixpkgs.config.allowUnfree = true;
```

this is highly recommended unless you only want non-proprietary packages.

### Edit the config file

Now is also a good time to add a few packages, create a user and uncomment the specified lines
for setting timezone, locale and things like that. No pressure, you can do this all later so keep
it minimal. You do not have to worry about mounting extra disks at this point.

Finally, execute:

```fish
nixos-install
```

if any errors pop-up, fix them now. This is imperative, especially if you are tweaking with the config file.
`nixos-install` should show no errors and should complete successfully.

and lets reboot:

```fish
reboot
```

You should be greeted with your bootloader, all the existing boot entries and a brand new option
for booting into Nix. Easy clap. You don't need to do any of that `edk-shell` bs.

## Post install steps

There is a quick and easy trick for setting up mounted disks in Nix. You _don't_ need to edit
`/etc/fstab` or `/etc/nixos/hardware-configuration.nix`

All you need to do is create the mountpoints you want with `mkdir` and then mount those disks
where you want them to live and then run `nixos-generate-config`

Don't worry this **_wont touch your existing config file_** it will only edit `/etc/nixos/hardware-configuration.nix`
and add your disks and mountpoints so that they are mounted on boot. Easy peasy.

## My experience

This was a fairly simple process, however on first boot I realized that my user didn't have a password set.
I couldn't login and I didn't know how to set the password. If this happens, just switch to another TTY using
`alt+F2` (or alt + any F3-F9) to drop into a TTY.

Login as root (the nix installer should have prompted for a password) and edit the `/etc/nixos/configuration.nix`
file and change user options.

For me, I ran

```fish
mkpasswd hunter2
```

and copy-pasted the password hash into my user password config

and then ran the following to set my user password:

```fish
passwd your_username
```

At this point I was able to login and start `Hyprland` using `greetd` and `tuigreet`

## Links

![Link to stackoverflow thread about Mac and Nix](https://superuser.com/questions/795879/how-to-configure-dual-boot-nixos-with-mac-os-x-on-an-uefi-macbook)

# Nix options for install

These are some options that you may consider using during the install step, or after on first boot.

### Boot options

I enabled systemd-boot and allowed nix to manage boot entries (this hasn't overwritten any existing boot entries)
I turned down the log level since I am using a TUI display manager and limited the amount
of Nix generations to keep in the boot menu to 9.

```nix
  boot.loader.systemd-boot.enable = true;
  boot.loader.efi.canTouchEfiVariables = true;
  boot.consoleLogLevel = 1;
  boot.loader.systemd-boot.configurationLimit = 9;
```

### User opts

create a user, add them to groups, set the default shell and packages.
Use `users.users.your_username`

(hash edited for brevity)

```nix
  users.users.sweet = {
    isNormalUser = true;
    extraGroups = [ "input" "audio" "video" "libvirt" "wheel" ]; # Enable ‘sudo’ for the user.
    hashedPassword = "$y$j9T$5...5xC"; # insert your hashed Password here
    shell = pkgs.zsh;
    packages = with pkgs; [ # these packages are available ONLY to your user
      firefox
      tree
    ];
  };
```

### General options that you will probably want to configure right away

allow the user to run random non-nix binaries.
set-up nix garbage collection to manage space

```nix
  programs.nix-ld.enable = true;
  programs.nix-ld.libraries = with pkgs; [ # check out src code for steam-run for an extensive list of LD libraries that let most binaries run
    stdenv.cc
    glibc
  ];

  nix.gc = { automatic = true; persistent = true; dates = "05:00:00"; options = "--delete-older-than 7d"; };
```

### Using greetd and Wayland

```nix
  services.greetd = {
    enable = true;
    settings = {
      default_session = {
        command = "${pkgs.greetd.tuigreet}/bin/tuigreet --time --cmd Hyprland";
        user = "sweet";
      };
    };
  };
```

### Nerd Fonts and Steam

```nix
  fonts.packages = with pkgs; [
    noto-fonts
    noto-fonts-cjk
    noto-fonts-emoji
    liberation_ttf
    fira-code
    fira-code-symbols
    mplus-outline-fonts.githubRelease
    dina-font
    proggyfonts
    jetbrains-mono
    ibm-plex
    maple-mono
    maple-mono-NF
    maple-mono-SC-NF
    iosevka
    fira-code-symbols
    nerdfonts
    powerline-fonts
  ];

  programs.steam = {
    enable = true;
    remotePlay.openFirewall = true; # Open ports in the firewall for Steam Remote PlayE
    dedicatedServer.openFirewall = true; # Open ports in the firewall for Source Dedicated ServerE
  };
```

### Hyprland

```nix
  xdg.portal.enable = true;
  programs.hyprland.enable = true;
  programs.zsh.enable = true;
  programs.thunar.enable = true;
```

#### Use the <nixos-unstable> channel

I'd recommend just jumping into flakes but if you just want to keep it simple you can stick with nix channels.
In my case the program `wl-gammarelay-rs` wasn't available on standard nixpkgs yet so I had to import the nixos-unstable channel
and import it into my packages. You can mix and match packages as you please:

create a file at `/etc/nixos/unstable.nix` and import it into `configuration.nix` or add it directly into your `configuration.nix` file.

```nix
{ config, pkgs, ...}:

let
  baseconfig = { allowUnfree = true; };
  unstable = import <nixos-unstable> { config = baseconfig; };
in {
  environment.systemPackages = with pkgs; [
    # add packages from the unstable branch
    unstable.wl-gammarelay-rs
  ];
}
```

make sure to actually add the unstable channel

```fish
sudo nix-channel --add https://nixos.org/channels/nixos-unstable nixos-unstable
sudo nix-channel --update

# rebuild
sudo nix-rebuild switch --upgrade

# check and see if <nixos-unstable> is added
sudo nix-channel --list
```

Anything else and you should be researching it first.
