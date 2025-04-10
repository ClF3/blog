---
title: "Compile and Install a Linux Kernel in Asahi Linux"
date: 2025-03-25T13:48:00+08:00
draft: false
tags: ["Linux", "Apple"]
categories: ["技术"]
---

> **TL;DR:** Install build tools, download the kernel source, copy `.config`, `make`, install everything, fix DTB path, run `update-m1n1`, **do not reboot without verifying DTB fix!**


## 1. Preparation

1. **Install Asahi Linux (if not already installed):**

    If Asahi Linux is not already installed, run the following command on macOS and follow the instructions to set it up.

    ```bash
    curl https://alx.sh | sh
    ```

2. **Download the kernel source:**

    Download the appropriate version from the [Fedora Asahi Repo](https://gitlab.com/fedora-asahi/kernel-asahi), then extract and enter the directory:

    ```bash
    tar -xf kernel-asahi-kernel-6.12.1-404.asahi.tar.gz
    cd kernel-asahi-kernel-6.12.1-404.asahi/
    ```

3. **Install the building essentials:**

    To compile the kernel, you’ll first need to install the required build tools:

    ```bash
    sudo dnf install make automake gcc gcc-c++ openssl-devel ncurses-devel flex bison
    ```

---

## 2. Config and Compile

1. **Copy the current kernel configuration:**

    Copy your existing kernel configuration as a starting point and then open it in menuconfig to adjust settings if needed.

    ```bash
    sudo cp /boot/config-$(uname -r) .config
    make menuconfig
    ```

2. **Compile the kernel:**

    Use the `-j` flag (adjust the number according to your CPU cores) for faster compilation:

    ```bash
    make -j 24
    ```

3. **Install the modules, device trees, VDSO and kernel:**

    Run the following command to install modules, device trees, VDSO, and the kernel itself:

    ```bash
    sudo make modules_install dtbs_install vdso_install install
    ```

But the installation is not finished yet. If you reboot now, you'll end up **unbootable**. Because after running `dtbs_install`, the device tree blob (dtb) files are installed in the wrong location.

---

## 3. Deal with DTB Problems

The default Makefile creates a symlink from `/boot/dtb` to `/boot/dtb-$(kernel-version)`, but actually installs the files in `/boot/dtbs/$(kernel-version)`. This mismatch makes the symlink invalid, causing the system to fail to locate DTB files at boot. Rebooting without fixing this will result in **missing peripheral drivers** and potentially an **unbootable system**.

To fix this, move the directory to match the expected name (replace 6.12.1 with your kernel version):

```bash
sudo mv /boot/dtbs/6.12.1/ /boot/dtb-6.12.1
```

---

## 4. Update m1n1

After every kernel upgrade, you need to update m1n1 to ensure the boot firmware is correctly aligned with your new kernel:

```bash
sudo update-m1n1
```

*Refer to the [Asahi Linux Docs](https://asahilinux.org/docs/alt/installing-gentoo/#updating-u-boot-and-m1n1) for more details on why this step is necessary.*

---

## 5. Done and Reboot

Before rebooting, **double-check that you have completed every step**. Skipping any part could result in an unbootable system or boot loops. A critical note: due to issues like the absence of USB keyboard support in GRUB (see [this issue](https://github.com/leifliddy/asahi-fedora-builder/issues/3)), recovering from such a scenario can be very challenging.

When you’re certain that everything is in place, you can safely reboot:

```bash
sudo reboot
```