+++
title = 'Vfio Passthrough Nixos'
date = 2024-08-07T21:58:07-08:00
author = 'sweetbbak'
+++

Easy VFIO Passthrough on NixOS

<!--more-->

Straight off the jump, if you have one GPU, this is not worth doing. I won't stop you but I will say that it is not worth it unless you are doing it purely out of curiosity and the willingness to learn.
Just dual boot at that point otherwise.

Im doing the same thing right now and ran into a whole mess of issues lol. But I’ve only been using Nix for a week or so. Its hard to do on any regular distro but it is doubly as hard on Nix ngl. But what I had to do was add:

IOMMU is a feature that you will need to enable that will allow you to passthrough your GPU to a virtual machine securely. You need a graphics card that has this feature
and you may need to enable this in the BIOS. The best way to find this out is to do some research on your GPU.

You can easily enable this in nix by adding this line to your nix configuration files:

```nix
boot.kernelParams = [
    intel_iommu=on
    iommu=pt
    vfio-pci.ids="10de:2489,10de:228b"
];
```

(where vfio-pci.ids are your GPU PCI ID's)

note the PCI ID’s “,” separated.
You can get this by running:

```sh
lspci -nn | grep -iE '(nvidia|amd)'
```

and finding your GPU. The Ids are in “[brackets:like_this]” You need both the GPU ID and the GPU Audio ID if you have it. It is a little hard to parse through visually...
Here is an example:

```sh
┌── sweet ─ ~sweetbbak.github.io    main
└─$ lspci -nn | grep -iE '(nvidia|amd)'
01:00.0 PCI bridge [0604]: Advanced Micro Devices, Inc. [AMD/ATI] Navi 10 XL Upstream Port of PCI Express Switch [1002:1478] (rev df)
02:00.0 PCI bridge [0604]: Advanced Micro Devices, Inc. [AMD/ATI] Navi 10 XL Downstream Port of PCI Express Switch [1002:1479]
03:00.0 VGA compatible controller [0300]: Advanced Micro Devices, Inc. [AMD/ATI] Navi 22 [Radeon RX 6700/6700 XT/6750 XT / 6800M/6850M XT] [1002:73df] (r
ev df)
03:00.1 Audio device [0403]: Advanced Micro Devices, Inc. [AMD/ATI] Navi 21/23 HDMI/DP Audio Controller [1002:ab28]
0a:00.0 VGA compatible controller [0300]: NVIDIA Corporation GA104 [GeForce RTX 3060 Ti Lite Hash Rate] [10de:2489] (rev a1)
0a:00.1 Audio device [0403]: NVIDIA Corporation GA104 High Definition Audio Controller [10de:228b] (rev a1)
```

this will blacklist the GPU from getting attached to drivers from what I understand. Its a pain though. Also I also used Astrid’s blog as reference but it’s important to note that this is not a full guide and does skip some important steps. She just shows some of the adapted steps in NixOS.

I ran into a ton of issues, like virt-manager crashing because it didn’t have a GTK cursor enabled… and virt-manager not being to access any directory besides “/” lol I might also do a write up because all the existing docs are essentially nil
edit: lspci I believe is in pkgs.usbutils or pkgs.toybox I think. You can use “nix search nixpkgs lspci”

script to test if your IOMMU grouping was successful:

```bash
#!/bin/bash
shopt -s nullglob
for g in $(find /sys/kernel/iommu_groups/* -maxdepth 0 -type d | sort -V); do
    echo "IOMMU Group ${g##*/}:"
    for d in $g/devices/*; do
        echo -e "\t$(lspci -nns ${d##*/})"
    done;
done;
```

[from Astrid](https://astrid.tech/2022/09/22/0/nixos-gpu-vfio/)

```nix
boot = {
  initrd.kernelModules = [
    "vfio_pci"
    "vfio"
    "vfio_iommu_type1"
    "vfio_virqfd"

    "nvidia"
    "nvidia_modeset"
    "nvidia_uvm"
    "nvidia_drm"
  ];

  kernelParams = [
    # enable IOMMU
    "amd_iommu=on"
  ] ++ lib.optional cfg.enable
    # isolate the GPU
    ("vfio-pci.ids=" + lib.concatStringsSep "," gpuIDs);
};

hardware.opengl.enable = true;
virtualisation.spiceUSBRedirection.enable = true;
};
```
