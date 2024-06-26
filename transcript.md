# Gentoo Installation Video Script

## Introduction

Hello there.

Today, we're going to install Gentoo, a source-based Linux distribution, on an encrypted BetterFS root.

The full list of packages which we'll install during this guide are in the description, split between 'Gentoo Base System', and our 'Sway Window Environment'. 

I've tried my best to streamline this guide to the absolute maximum without sacrificing on explaining what's actually going on, so even someone new to Linux can _hopefully_ understand it. But if you're curious about any of the packages or `USE` flags we've set during this guide, you're going to have to look those up yourself. Which I do show you how to do.

The goal is to create a lightweight, heavily optimized base system consisting entirely of binaries we compiled ourselves. If you're not interested in installing a desktop environment, don't worry: you'll just finish the video sooner than everyone else.

Installing Gentoo might seem a little daunting, but I promise you can do it. While the entire installation could take many hours,
the vast majority of that time is waiting for software to compile; so you can expect to put in about an hour of actual effort.

That's enough rambling; let's get started.

## Media Creation Redirect

First, we need to create some Gentoo installation media. The instructions differ slightly for Linux and Windows users, so we're going to hop over to a another video for a few minutes. Both videos are linked in the description, but Linux users can click the card in the top right of the screen now, and...

Now Windows users can do the same.

## Booting

Once you have your installation media prepared, you can insert it into your target machine, turn the computer on, and press the hotkey to open the boot menu. What that hotkey is will depend on your computer, though usually it's one of the function keys from F8 to F12. There's a table on screen, but if it doesn't cover your computer's manufacturer, or doesn't work for you, you can either search online, or mash the funciton, delete, and escape keys until you get into the boot menu, or the BIOS menu, where there should be a 'Boot' tab.

Once you're in the boot menu, select your boot media. It'll probably have `USB` in the name, and if there are two options, one of which is `UEFI`, select that one. You should see a Grub menu which looks like this.

## Keyboard Layout Selection

Hit enter, and after a short wait, you'll be prompted to select your keyboard layout. Those with a US keyboard layout can just hit enter, but the rest of us will need to type a number. For me that's `42`, for UK.

## Disk Partitioning

After another short wait, you'll be dropped into a terminal. The first thing we need to do is work out the identifier for the drive we want to install Gentoo onto. We'll type `lsblk` and hit enter to list block devices, which is a fancy way of saying storage devices, then look for a device with the correct size. It's likely to be something like sda or sdb, or nvme0 or 1. We're going to overwrite this drive, so make sure you don't have anything important on it.

Now, we'll type `parted /dev/`, then whatever the name of your drive is. Mine is `sda`. Parted is a disk management utility, and we're going to use it to partition our drive in two parts: a small boot region, and an encrypted BetterFS root which will host our system.

First, we need to type `mklabel gpt`, to create a new guid partition table. This is a small data structure which describes the partitions on the drive. You'll need to confirm with `Y` if you've used this drive before.

Next, we'll run `mkpart`, short for make partition,` boot fat32 0% 128M`. This will create a fat32 partition with the label `boot`, which starts at the beginning of our drive, and ends 128 megabytes from the start.

Then we'll run `mkpart root btrfs 128M 100%`. You could change 100% for any value you like, such as `50G` for a 50 gigabyte root, or `50%` to use half of the drive's space.

Now, we'll run `set 1 boot on`, to mark the first partition as bootable, `p` to print out our partition table and check we got it right, then `q` to exit parted.

Congratulations; you just partitioned a disk. Let's run `lsblk` again, and you should see two new entries for the partitions you just created. Take note of their identifiers, which will be something like `sda1` or `nvme0p1`.

## Root Encryption

Now we need to encrypt our root partition. We'll run `cryptsetup luksFormat -s256` for a 256 bit key,` -c` for cypher` aes-xts-plain64 /dev/` then the name of the second partition, like `sda2`.

You'll need to confirm by typing `YES` in all capitals, before setting and verifying an encryption passphrase for your disk. Make sure to use something strong.

Next, we'll run `cryptsetup luksOpen /dev/` the name of that partition again,` cryptroot`. This will open the encrypted patition we just created, so we'll need to enter the passphrase again.

Now we've set up our encryption, we can create our filesystems.

## Filesystem Creation

First, run `mkfs.vfat -F 32 /dev/` then the name of your first partition, like `sda1`. Once that's done, run `mkfs.btrfs -L BTROOT /dev/mapper/cryptroot`. When we ran `luksOpen`, cryptsetup mapped our decrypted partition to that name, which is how we'll be referring to it most of the time going forward.

## Mounting & Subvolume Creation

Now, we're going to mount our root partition, so we can use it. Run `mkdir`, short for make directory,` /mnt`, short for mount, `/root` to create a directory we can mount the parition to, then run `mount -t`, type,` btrfs -o`, options` defaults,noatime`, because we won't track file access times; there's no reason to do so, `,compress=lzo,autodefrag /dev/mapper/cryptroot /mnt/root`.

Now we've mounted our root partition, we can create a couple of subvolumes. On other filesystems, like `ext4`, it's common practice to create separate partitions for various parts of your Linux system, such as `root` and `home`. With BetterFS, subvolumes achieve the same thing. Run `btrfs subvolume create /mnt/root/activeroot`, then `btrfs subvolume create /mnt/root/home`.

Now we'll mount the `activeroot` subvolume by pressing up until we get our last `mount` command back, adding `subvol=activeroot` to our options, and changing the mount directory to `/mnt/gentoo`. Then we'll `mkdir /mnt/gentoo/home`, and mount by going back to our last `mount` command, and changing `activeroot` to `home`, and tacking `home` to the end of our mount directory.

Now we'll run `mkdir /mnt/gentoo/boot` for a mountpoint for our boot partition, and `mkdir /mnt/gentoo/efi` to mirror it. Both directory names are idiomatic, with `boot` being more common, but some poorly written scripts and software might favour one over the other. With both directories created, we'll run `mount /dev/sda1 /mnt/gentoo/boot`, with `sda1` being _your_ first partition. Then, the same command again but for the `efi` directory.

## Internet Connection

Ok, we're done preparing our disk, so now we need to connect to the internet. If you're using ethernet, a wired connection, you shouldn't need to do anything. If you're on WiFi, however, we need to connect to a network. Run `wpa_cli`, then `scan`. This should scan for networks in range, and we can run `scan_results` to print a table of available networks.

Now, we'll run `add_network`, which should print out a number, which'll probably be `0`. We need to set the `ssid`, or name, of the new network slot, so we'll type `set_network 0`, or whatever that number it printed out was,` ssid "` then the name of the network, which is the last column in the table of scan results. Close the double quotes, and hit enter, before running `set_network `, whatever the number was,` psk "`, and then type the password for the network, close the double quotes, and hit enter. Now we'll enable that network by running `enable_network `with the same number again, before running `save_config`, and `quit` to exit.

## Time Sync and Stage3 Download

Once we're connected to the internet, we'll run `chronyd -q` to sync to network time, then `links https://distfiles.gentoo.org/releases/amd64/autobuilds` to open a text-based web browser to Gentoo's website.

Hit enter to close the welcome message, and once the page loads, you should see a list of files and folders, the first several of which are named for timestamps. Use the arrow keys to scroll down to the last of the timestamp folders, which will be the latest build, and hit enter to open it. There'll be another list of files, and we're going to scroll down until we find a file whose name starts with `stage3-amd64-hardened-openrc`.

There'll be several files of the same name, with differing file extensions. We'll highlight the first, which is a `.tar.xz` file, and press `d` to download it. We can hit `Enter` a couple of times to send that download into the background, before scrolling down a couple of files to the `.asc` file, and hitting `d` to download that. It should take a second or so to download, at which point we can hit `Escape`, use the right arrow key to highlight the `Downloads` menu, then hit `Enter` a couple times to bring back the first download. Once it finishes, the modal should disappear, and we can hit `q`, then `Enter` to exit Links.

## Stage3 Verification & Extraction

Back in our terminal, if we `ls`, it should list both files. We'll run `gpg --import /usr/share/openpgp-keys/gentoo-release.asc` to import the Gentoo releases signing key, before running `gpg --verify` on the `.asc` file we downloaded. We should get `Good Signature`. If you don't, try downloading the same two files from previous snapshots.

Now we can `rm` (remove) the `.asc` file because we're done with it, before running `mv ./stage3`, hit tab to autocomplete to the `.tar.xz` file, then type ` /mnt/gentoo` to move the file to the root of the Gentoo system we're building. Then we'll type `cd /mnt/gentoo` to change our terminal's working directory.

It's time to unpack our Stage3. This archive contains an absolutely minimal base Gentoo system which we can use to configure, compile, and install all of our own software. It does, necessarily, contain many binaries, but we'll recompile them at the first opportunity. We'll type `tar xpvf ./stage3`, hit tab to auto-complete, then` --xattrs-include="*.*" --numeric-owner`, and hit enter to start unpacking.

## Pre-Chroot Configuration

Once the tar file is unpacked, we can `rm` the `stage3` tar file, and `ls` to see a directory structure which will be extremely familiar to anybody who's ever used Linux before.

### Locale Configuration

#### locale.gen

Our next task is configuring our system. First, we'll run `nano ./etc/locale.gen` to open a text editor, where we'll insert the locales we want our system to support. I'll add `en_GB ISO-8859-1` and `en_GB.UTF-8`. You'll probably want the same two variants, but a different locale, and maybe a different language. We can press `Ctrl + S` to save, then `Ctrl + X` to close Nano.

#### locale.conf

Now we need to create a new file. Run `nano ./etc/locale.conf`, and add `LANG="en_GB.UTF-8"` on one line, substituting the locale for whichever one you used, then `LC_COLLATE="C.UTF-8"` on another line.

### Hardware Clock

Next, we'll `nano ./etc/conf.d/hwclock`. You _probably_ don't need to change this file, which configures the hardware clock for your computer, but if you're dual booting with Windows, you'll need to replace `UTC` with `local`.

### Keymaps

Now we'll open `./etc/conf.d/keymaps`, where we'll change `us` to whatever keyboard layout we need. For me, that's `uk`.

If you're not sure what to write here, we can switch to another terminal by pressing `Alt + Right`, then run `ls /usr/share/keymaps/i386/qwerty`, assuming you're using a QWERTY keyboard, which will list all of the available keymaps of that type. There's a `uk.map.gz` file, which is mine, but you can use any of the options here. Just remove the `.map.gz` from the name, and that's the value you'll input.

Switch back to the previous terminal with `Alt + Left`, and once we've set the value, we can save and exit.

### Timezone

Now we'll set our timezone by running `echo "Europe/London" > ./etc/timezone`, substituting in your timezone.

### Filesystem Table

Next, we're going to create an `fstab`, or filesystem table. This file tells Linux how to mount all your drives whenever you boot. We'll run `nano ./etc/fstab`, use `Alt + Delete` to delete all the lines except the column headers, then tell Linux to mount the device with `LABEL=BTROOT` to `/`, with a filesystem of `btrfs`, options of `default,noatime,compress=lzo,autodefrag,discard=async,subvol=activeroot` and` 0 0`. Then we'll do the same thing, but for `/home`, on the `home` subvolume.

Now we'll hop back into another terminal with `Alt + Right`, and run `blkid`, or block id, for ` /dev/sda1`, or whatever the identifier for your boot partition is. We can switch between the two terminals to type `UUID=`, then whatever the universally unique ID for your boot partition is, then` /boot vfat umask=077 0 1`. Then we'll write the same line again, but mounting to `/efi`.

It's very important you get this file correct, so go ahead and pause, and double and triple check everything.

### Grub Configuration

Now we're going to write a configuration file for Grub the bootloader, which is the program which actually launches Linux when you turn on your computer. We'll need to tell Grub how to decrypt our root, so we'll run `nano ./etc/default/grub`, and write the lines `GRUB_DISABLE_LINUX_PARTUUID=false`, `GRUB_DISTRIBUTOR="Gentoo"`, and `GRUB_TIMEOUT=3` to set the timeout of the Grub menu before it defaults to the first option to 3 seconds. You can set this however you like, but I'd recommend 2 seconds as the minimum. The default is 5.

Now we'll write the lines `GRUB_ENABLE_CRYPTODISK=y`, and `GRUB_CMDLINE_LINUX_DEFAULT="crypt_root=UUID= quiet"`. We need to get the partition UUID for our root partition, so over in another terminal we'll run `blkid /dev/sda2`, or whatever your root partition's identifier was, then copy the UUID.

Again, this file is very important, and if you don't get it right, it will be a hassle to fix. Pause, and check it all.

### Portage Configuration

We're almost at the point where we can enter our new Linux system and start compiling all our software, but there's one last job to do: configuring Portage.

Portage is the package manager for Gentoo; its equivalent to `apt`, or `pacman`, or any one of the million other package managers out there. For Windows users, package managers are like the Microsoft Store, but if you _actually wanted_ to use it.

Portage is the magic of Gentoo. Most package managers just download generic binaries somebody else compiled, and we have to trust that they compiled those binaries from the source code we have available to us. There's a card in the top right of the screen to a short video if you don't understand why that's nonideal.

Portage, on the other hand, downloads the source code, and compiles it. Not only does that mean that ultimately, we only need to trust the compiler we started with, as far as binaries go, but we can optimize the programs we install for our specific processors, and our specific use-cases. We're going to be compiling with all the optimization we can do, which will net us faster execution times, and therefore less power consumption, less heat, and better battery life, if we're on a battery.

So to get started, we're going to run `rm -rf ./etc/portage/package.use`, `rm -rf ./etc/portage/package.license`, and `rm -rf ./etc/portage/package.accept_keywords`. Then we'll `mkdir ./etc/portage/repos.conf`, and `cp` (copy)` ./usr/share/portage/config/repos.conf` to `./etc/portage/repos.conf/` with a new name of `gentoo.conf`. This file tells Portage where to get its list of packages.

Rather than have you type out every single line of the Portage files manually, I've created some templates we can download. I will, however, explain what the files are, and what each line is achieving, and how to look up more information so you'll be able to modify these files to suit your own needs either now, or in the future. Portage is the beating heart of Gentoo, and you _will_ need to modify these files if you use this system.

So, we'll run `wget https://github.com/libreisaac/portage-config/archive/refs/heads/main.zip` to download my portage configuration templates from GitHub.

Then we'll run `unzip ./main.zip`, `rm ./main.zip`, and `ls -a ./portage-config-main` so you can see all of the files, including hidden files, if there were any, which you're about to add to your system.

Next, we'll move all the files in that directory with `mv ./portage-config-main/* ./etc/portage`, and, now that the directory is empty, `rm -r ./portage-config-main`, before running `mkdir ./etc/portage/env`, and `mv ./etc/portage/no-lto` to` ./etc/portage/env`.

Now, we can start looking at these files; there will be things for you to change and again, it's crucial you understand them.

#### make.conf

First, we'll open `make.conf`. This file is perhaps the most important, setting global variables for every single package we compile.

There are two main things we're configuring: `USE` flags, and compiler flags. `USE` flags enable or select features in a package, deciding which code we should compile. Most `USE` flags, when set, will result in more code being compiled. Compiler flags tell the compiler itself _how_ to compile each package, and they are where most of the optimization comes from.

The first thing this file does is clear all of the default `USE` flags.

Then, we tell Portage to only accept free, as in freedom, not cost, because none of the packages you'll be installing via Portage cost any money, software licenses by default.

Next, we set up our compiler flags. The `COMMON_FLAGS` set an optimization level of 3, which is the highest, tell the compiler to `pipe` output, which uses more memory during compilation, but speeds up compilation overall, compile for the `native` architecture, meaning our exact CPU, and perform link-time optimization across seven processes.

Link-time optimization can yield multiple-digit percentage performance improvements for many programs, but it doesn't _always_ work. The `WARNING_FLAGS` convert several `LTO`-related warnings into errors, because they otherwise wouldn't fail compilation, we wouldn't see them, and they'd almost certainly result in broken behaviour at runtime.

These flags all apply across multiple programming languages, as opposed to the `RUSTFLAGS`, which are specific to the Rust programming language. Similarly, we compile for the native CPU with an optimization level of 3.

`MAKEOPTS` defines the number of processor threads to use when compiling. To work out the correct number, take half the number of gigabytes of memory you have, and compare it to the number of threads your computer has, using whichever is smaller. You can run `free --giga` to get the amount of memory you have, and `nproc` to get the number of threads. My VM has 16 threads and 32 GB of memory; half of 32 is 16, which is the same, so I use 16. If I had 30 GB of memory, I'd use 15 threads, and if I had 34 GB of memory, I'd still use 16.

Below, we set all of the `USE` flags we want to apply to all packages by default. There are too many for me to through every one in this video, but I'll show you how to look up `USE` flags as well as packages shortly, so you can investigate for yourself.

The `VIDEO_CARDS` flag is used to indicate what graphics card manufacturers should be supported. If you have an AMD GPU, it will be `amdgpu radeonsi`. For Nvidia, `nouveau`. For Intel, `intel`, though if you have something as old as, or older than, the Intel HD 3000, it'll be `intel i915`. You can set as many of these as you like, so if you have GPUs from multiple manufacturers, that's not an issue. I'm running in a VM, however, so I'll go with the generic `vesa`.

`CURL_SSL` selects an `SSL` library for the program `curl`, which makes web requests, and is called out to by many other pieces of software.

`PAX_MARKINGS` is for Python security hardening.

Below that, we set target programming language versions. Various packages can be compiled for different versions of the same language, but it's a good idea to optimize for both consistency, and recency.

#### no-lto

For packages which don't successfully compile with `LTO` enabled, we can manually disable `LTO` with an environment file. If we open `./etc/portage/env/no-lto`, you'll see a file which sets the common flags again, without `LTO` or its `WARNING_FLAGS`, and which subtracts `lto` from the global `USE` flags.

Packages which are configured to have this compilation environment have the `make.conf` applied to them, _then_ this environment file afterwards, which allows us to overwrite environment variables without having to duplicate everything we don't want to change.

#### package.env

To tell a package to use a custom environment, we add a line to `./etc/portage/package.env`. The `cmake` and `slang` packages are used in the build toolchain of most packages, but unfortunately, didn't successfully compile with `LTO` when I tried; so we apply the `no-lto` environment to them.

#### package.use

In the `./etc/portage/package.use` file, we can set `USE` flags for specific packages. These `USE` flags get applied in addition to the default `USE` flags defined in our `make.conf` file, and any environment applied to the package, but we could remove a default `USE` flag with a `-`, like `-lto` in our `no-lto` environment.

You may notice the commented out line `sys-kernel/linux-firmware redistributable`. The `linux-firmware` package contains drivers, and can optionally include proprietary binary blobs whose source we can't interrogate, or compile for ourselves. It used to be pretty much necessary to use binary redistributables, but a lot of systems can get away without them nowadays. You could uncomment this line, but I think it's worth trying without.

#### package.license

In the `package.license` file, we can add or remove license acceptances for specific packages, just like the `package.use` file allows for `USE` flags. Again, we have a commented out line here for `linux-firmware` to accept binary redistributable licenses. If you want to install binary redistributable drivers, this line will need to be uncommented as well as the `package.use` line.

#### package.accept_keywords

The `package.accept_keywords` file can be used to enable installing unstable versions of a package. Portage won't install unstable versions by default, for good reason, but sometimes you want to test a new version of a package, or install a package which is entirely marked as unstable.

Don't change this file unless you know what you're doing, but we _are_ going to allow installing unstable versions of the package `tuigreet`, because the entire package is marked as unstable. I've been using it for a long time now, and never run into any problems.

#### Mirrorselect

Now, we're going to select mirrors, or servers, to download portage package information from. We'll run `mirrorselect -i` for interactive,` -o` for output, and append that output to the end of our `make.conf` with` >> make.conf`. If you only do one arrow, it will overwrite the `make.conf`, so don't do that.

A text-UI will open where we can navigate with the arrow keys, `Page Up` and `Page Down`, and we can select mirrors with `Space`. I'm going to select `https` and `rsync` servers for Portugal and the UK. You can select servers near you, again, preferably with at least one `https` and one `rsync` option.

Once we've selected our mirrors, we'll hit `Enter`, then run `nano ./etc/portage/make.conf` to check we did it right. I'll also add a comment to this new line at the bottom; I recommend you comment all your own changes to these configuration files for future reference.

## Package and `USE` flag lookups

To look up packages and `USE` flags, we can go to `packages.gentoo.org` in a web browser. 

On the index page, there's a search box we can use to look up packages; let's look for `ffmpeg`.

On the package page, we see a list of versions, green being stable, yellow unstable, a link to the upstream, where the source code lives, a list of local and global use flags, and on the right, links to documentation, forums, and the ebuilds repository where the glue between the package's build scripts and Portage lives.

The `USE` flags are all links, and we can hover over or click them for more detauls.

If we press `USE flags` in the navbar, we'll see a nice little visualization of the most common `USE` flags, the larger circles being used in more packages. These are all links.

We can also search for `USE` flags on the `Search` page, view a list of all global `USE` flags, all local `USE` flags, and all `USE Expand` flags.

The distinction between global and local `USE` flags is somewhat arbitrary; we can set either in our `make.conf`, or `package.use` on a per-package basis, but global `USE` flags are more likely to live in our `make.conf`.

`USE Expand` flags cover flags like `VIDEO_CARDS`, and we can see every allowable version for them on this page.

If we search for the `ffmpeg` `USE` flag, we'll see a list of packages which define it as a local `USE` flag, as well as those which define it as a global flag. While `ffmpeg` is a package itself, it's also a `USE` flag because certain packages can optionally call out to it to perform various tasks.

## Chroot

Now, there's just a few commands to run before we compile all our software. But those commands, and compiling all our software, need us to be in our new Gentoo environment.

It's not ready for us to actually boot into it yet, but we can perform a `Change Root` from a Live CD to enter it before it's complete.

First, we need to copy `/etc/resolv.conf` into `/mnt/gentoo/etc/`, so we'll still be able to talk to the internet once we change root.

Now I'll write out some commands for you to copy once I've explained them.

```
mount --types proc /proc/ /mnt/gentoo/proc
mount --rbind /sys/ /mnt/gentoo/sys
mount --rbind /dev/ /mnt/gentoo/dev
mount --bind /run/ /mnt/gentoo/run
mount --make-rslave /mnt/gentoo/sys
mount --make-rslave /mnt/gentoo/dev
mount --make-slave /mnt/gentoo/run
chroot /mnt/gentoo /bin/bash
source /etc/profile
export PS1="(chroot) ${PS1}"
```

The mount commands temporarily mount some folders from the live CD to folders in our Gentoo installation, so we can have access to various binaries which don't exist in our Gentoo environment yet, then the `chroot` command changes the root to `/mnt/gentoo`, and runs `/bin/bash`, which is the terminal you've been using this whole time.

The `source /etc/profile` command loads some default environment variables in the `chroot` terminal, and the `export PS1` command changes the little tag you get before each command you run, to clearly identify terminals which we've `chroot`ed; we can still change between terminals, and all the other terminals are still in our Live CD environment, so we want to make sure we're running future commands in the right place.

Go ahead and pause, and copy those commands down.

If we wanted to `chroot` in another terminal, we'd just run the `chroot`, `source` and `export` commands again; the mounts only need to happen once.

## Portage Sync and Configuration Application

Now, we'll apply some of the configuration files we created earlier. First, we'll run `emerge-webrsync` to sync portage from our seleted mirrors.

Next, we'll run `emerge --sync --quiet`.

Then, we'll apply out timezone configuration with `emerge --config sys-libs/timezone-data`.

We'll run `locale-gen` to generate the locales we defined, then `env-update` to update the environment variables in this terminal with all the newly configured options. We'll need to re-run the `source /etc/profile` command, and re-export our `PS1` variable, which was just cleared.

## CPU Flags

Now, we can emerge our first package. Run `emerge app-portage/cpuid2cpuflags` to emerge a small package which will read the `cpuid` variable which is made available by your computer's firmware, and tell us which `CPU_FLAGS` we can set. CPU flags are another type of `USE` flag which tell packages they can use certain CPU instruction sets, mostly to the benefit of performance.

Once `cpuid2cpuflags` has compiled, we'll run it. It'll spit out some `CPU_FLAGS`, and we'll open our `make.conf` in another terminal, and add `CPU_FLAGS_X86` to our compilation option section, then write out all the `CPU_FLAGS` by hopping between the two terminals. You'll need to base yours on the output _you_ got, not the output I got.

## Global Recompilation

Now it's time to recompile all of the binaries we got in our Stage3. Because we didn't compile them, none of our optimization configuration has been applied to them, and we're trusting whoever compiled those binaries to not have lied to us.

To recompile everything, we'll run `emerge --emptytree -a -1 @installed`. 

Now at some point, when emerging packages, you are going to encounter an error. It could happen during this guide. It could happen a year in the future. Probably not that long, though.

There are two main categories of errors you will encounter: there are portage errors, and there are compilation errors.

Portage errors happen before any package starts compiling, and they're extremely descriptive.

If it says something about `USE` flags, you're going to open your `package.use` and change your `USE` flags.

If it says something about a package license, you're going to open `package.license` and change that.

If it says something about an unstable package, assuming you know that it's definitely safe to use that unstable package, you're going to be changing your `accept_keywords` file.

Compilation errors happen after packages start compiling, and are much more difficult to troubleshoot. If you're emerging multiple packages at once, it's probably a good idea to try emerging them one at a time. It might be that one package just doesn't emerge until some other package is emerged before it, because it has a dependency that it doesn't declare properly.

Failing that, it will _always_ tell you a location for a build log, which you can open, and read, and try to figure out what's going on. If you can't figure out what's going wrong, you're going to google the error messages you get. And if _that_ doesn't help, then you're going to go to a forum, or the comments on this video I guess, and ask for help.

This is going to take a while, potentially serveral hours, so go take a break, touch grass, and find something else to do until it completes.

## Rust

Once we've recompiled everything, we're going to compile the Rust programming language's compiler, which will be used to compile many user-facing programs. We'll run `emerge dev-lang/rust`, which again, will take a while. The Rust compiler is one of the slowest single packages to compile; while recompiling all of the initial binaries took about TODO hours for me, Rust takes about TODO minutes.

Once Rust has finished compiling, we'll open our `package.use` file, and add the `system-bootstrap` flag to `rust`. In the future, the `rust` compiler will be compiled with the Rust compiler we just compiled and installed.

## Emerge Base Packages

Now, we're going to install all the base packages we want for our system. I'll type them out, you can pause, check them over, then copy them down.

```
emerge --ask \
    sys-kernel/gentoo-sources sys-kernel/genkernel sys-kernel/installkernel sys-kernel/linux-firmware \
    sys-fs/cryptsetup sys-fs/btrfs-progs sys-boot/grub sys-apps/sysvinit \
    sys-block/parted sys-auth/seatd sys-apps/dbus sys-apps/pciutils app-admin/sysklogd sys-process/cronie net-misc/chrony \
    net-misc/networkmanager app-admin/doas app-shells/bash-completion dev-vcs/git \
    gui-libs/greetd gui-apps/tuigreet app-editors/nano
```

For me, all of these packages together take about as long as Rust took to compile. Go ahead and pause, and emerge those programs.

## Doas Configuration

Once all of our initial packages are compiled and installed, we'll create a `doas` configuration. `doas` is like `sudo`, or `run as admin` in Windows.

We'll run `nano /etc/doas.conf`, and write `permit :wheel` to allow users in the `wheel` group to use `doas`, save and exit, then run `chown -c root:root /etc/doas.conf` to make the root user account the owner of that file, and `chmod -c 0400 /etc/doas.conf` to make that file readonly.

## Greetd Configuration

Next, we'll open `/etc/greetd/config.toml`, where we'll change `vt` to `current`, and the command `greetd` runs for our login screen to `tuigreet`. We'll change the command `tuigreet` runs from `sh` to `bash`, and add `-t` to show the time on our login screen.

Then we'll run `usermod greetd -aG video` to add the `greetd` user account to the `video` group. Then we'll run the same command again, but for the `input` and `seat` groups.

### inittab

Next, we're going to start `greetd` on boot. There are several ways we can achieve this, but in my view, the simplest and cleanest is to modify our `/etc/inittab` file, commenting out the existing `agetty` lines, and adding a single line for `greetd`.

Once `OpenRC`, our service manager is finished initializing our system when we boot, terminals are spawned by these lines for us to log in and run programs from. If you're installing a desktop environment, having multiple terminals doesn't make much sense. If you're not going to install a desktop environment, you may want to replace all of the terminals we commented out.

## Service Configuration

Now, we're going to run `rc-update add seatd boot` and `rc-update add dbus boot` to add the `seatd` and `dbus` services to our `boot` run level. These services are essential for user login sessions, and you can think of the run level as the basic order in which services are started; we want these to run when we're booting up.

Then we'll run `rc-update add NetworkManager boot`, to add the network manager service. This one doesn't necessarily need to run at boot, and could run at the `default` level or even, if you were to set up a login run level, at login time.

Finally, we'll run `rc-update add sysklogd default`, `rc-update add chrony default`, and `rc-update add cronie default` to enable our system logging, time synchronization, and scheduled task services.

### Hostname

Now we'll run `rc-update delete hostname boot`, to remove the default service which sets the system's hostname. We're using `NetworkManager`, which will be responsible for that, so we'll run `rc-service NetworkManager start`, then `nmcli general hostname my-computer`, to set the hostname to `my-computer`. Obviously, you can set it to whatever you like.

## User Management

Next, we're going to manage the user accounts on the system. First, we'll run `passwd` to set a password for the root account.

Then we'll run `useradd isaac`, replacing `isaac` with your own name, to create the user account we're going to actually use. We'll run `passwd isaac` to set the password for that new account, then `usermod isaac -aG users` to add that account to the `users` group. We'll run that same command again for the `wheel`, `disk`, `cdrom`, `floppy`, `audio`, `video`, `input` and `seat` groups.

## Kernel Compilation

Now, it's time to compile our Linux kernel. The kernel is the brain of the operating system. We'll run `eselect kernel set 1`, to tell Portage we want to compile the kernel ourselves, and run `genkernel --luks`, to support disk encryption, ` --btrfs`, to support our BetterFS root, ` --keymap`, to enable cryptunlock keymapping, in case you can't type your encryption passphrase on the default keyboard layout at boot time, `--no-splash`, because we're not going to show an extra splash screen at boot time, `--oldconfig` to load any existing kernel configuration, `--save-config` to save the configuration we make for our kernel, ` --menuconfig` to open a menu for us to configure our kernel with, and ` --install all` to install the kernel. Because I'm in a virtual machine, I'm also going to add ` --virtio`; you shouldn't add this unless you're also installing Gentoo into a VM.

After a short wait, you should see a text UI menu like this. You can configure your kernel in a million ways, with many tradeoffs and improvements for your specific use-case, but today, we're going to just change three settings. In the future, I may create a video on complete kernel configuration, but it's a massive topic. You can replace your system's kernel at any time, so don't worry about configuring it right away.

Today, all we'll do is go to the `Filesystems` option, scroll down to `BTRFS`, hit `Space` to switch it from a module to a builtin, then `Escape` to go back, go to the `Cryptographic API`, then `Length Preserving Cyphers`, where we'll make `XTS` a builtin.

Finally, `Escape` a couple times, down to `Gentoo`, `Init Systems`, and disable `systemd` by highlighting it and pressing `Space`. Now we can hit `Escape` a few times, save our configuration with `Enter`, and the kernel will start compiling. For me, it takes about 15 to 20 minutes.

## Grub Installation

Now our kernel is compiled, there's one thing left to do before we can boot into our new Gentoo environment; we need to install the bootloader to actually boot it! To do that, we'll run `grub-install --target=x86_64-efi --efi-directory` is `/boot --removable --recheck`, then `grub-mkconfig -o` is `/boot/grub/grub.cfg`.

Finally, we can run `reboot` to reboot our computer. You'll probably want to remove your installation media at this point.

## First Boot

First, you should see a familiar Grub menu, only this time, the options are for `Gentoo`, and not a `Gentoo Live CD`. We'll hit enter to select the first option, then be prompted to enter our encryption key. Once we've unlocked our drive, we'll see some OpenRC output as all of the services are launched, then get dropped into Tuigreet.

We can log in as our main user account, and run `doas -u root passwd -l root` to lock the root user account. If you're on Wifi, you also might want to run `nm-tui`, which will open a small text UI where you can select and connect to a network.

The eagle-eyed among you may have noticed an error message during the OpenRC stage of booting. If we run `rc-status`, we can see the culprit: our logging service isn't running. This is because the service is configured to rely on the `hostname` service, which we removed. We need to open `/etc/init.d/sysklogd` as root, and comment out the `hostname` dependency. If we save, exit and reboot, we won't see that same error message during OpenRC initialization and, once we log in, if we run `rc-status`, the logger service is running.

Congratulations, you've installed Gentoo! Most of you will probably want a graphical interface with windows and pretty UI, however, so let's add one.

If you don't want a GUI, or you want to install your own, click the card in the top right of the screen to get the playlist of videos in this series. While future videos will be using my Sway environemnt, they aren't dependent on it, and many don't even need a desktop environment.

## Sway Compilation

To build a desktop environment using Sway, we're going to first download some configuration files similar to what we did for our portage configuration.

Run `wget https://github.com/libreisaac/sway-config/archive/refs/heads/main.zip`, then `unzip main.zip`, `cd` into `sway-config-main`, and `ls -a` to see what's in here.

First, there's a `sway.sh` script which we can view with `nano`.

This script checks whether the `XDG_RUNTIME_DIR` environment variable has been set. This variable would usually be set by a more heavy desktop manager, but we're going lightweight, so we'll have to set it ourselves.

If the environment variable isn't set, we set it based on our user ID, and create a temporary directory for the current login session.

At the end, we run Sway through a `dbus` session, which is necessary for Sway to run properly.

To use this script to launch Sway, we'll enter a root terminal with `doas -u root /bin/bash`, then `mv` the `sway.sh` script to our `/etc/greetd` folder, then `nano /etc/greetd/config.toml`, where we'll change the command `tuigreet` runs on login from `/bin/bash` to our `/etc/greetd/sway.sh` script.

If we `ls -a` again, you'll see there's a `package.use.append` file. If we open it in `nano` to see what it contains, it sets up `USE` flags for Sway, Waybar, Suspend, which is a package which allows us to send the computer to sleep, Firefox, IMV, which is an image viewer which can be launched from the terminal, and MPV, which is a video player which can be launched from the terminal.

We'll append these `USE` flags to our `package.use` file by running `cat package.use.append >> /etc/portage/package.use`; make sure to use two arrows to append, rather than overwriting!

Now we can `rm ./package.use.append`. The only thing remaining is the `.config` folder, which we'll move into our home directory with `mv .config ~/`.

Now we can `cd ../` to go back one directory, and `rm ./sway-config-main -r`, since that directory is empty now. 

### Emerge Main Packages

Now it's time to emerge the packages for our Sway environment. I'll type the command out quickly...

```
emerge --ask sway waybar swaylock swayidle swaybg suspend foot ranger cmus htop grim slurp wl-clipboard alsa-utils fontawesome imv mpv pipewire wireplumber
```

And now you can pause and copy the command. These packages will take a little while to compile, so go find something to distract yourself with.

### Emerge Bemenu

Once those packages are done compiling, we're going to run another `emerge --ask`, this time just for `bemenu`. The dependency tree for `bemenu` is slightly broken, so we had to emerge it after `Sway`. This shouldn't take very long at all.

Now we've emerged and configured everything, we can `reboot` to restart our computer, and when we log back in, we should end up in Sway!

## Firefox

We can open a terminal by pressing `Alt + T`, then run `nano ./.config/sway/config`. This is our Sway configuration file. The first thing you'll want to do is change the `xkb_layout` under `Input Configuration`, save with `Ctrl + S`, then reload Sway's configuration file by pressing `Alt + Shift + C`.

Next, you may want to change the modifier key for all the keyboard shortcuts this configuration file defines. The default, `Mod1`, is `Alt`. I prefer `Mod4`, which is the `Windows` key, but I can't use it in my VM. Again, save and hit `Alt + Shift + C` to reload your Sway configuration. From now on, I'll be describing hotkeys with `Mod` as a stand-in for `Alt` or the `Windows`, `Super` or `Command` key depending on what you configured.

### No-LTO

Now, we'll use `Mod + 2` to switch to a different workspace, open another terminal with `Mod + T`, run `doas -u root nano /etc/portage/package.env` and add `www-client/firefox no-lto`; when I tried, Firefox failed to compile with `LTO` enabled.

### Emerge

Now we can emerge Firefox with `doas -u root emerge --ask firefox`. This will take a long time, but rather than wait for it to compile, we'll hop back over to our first workspace with `Mod + 1`.

### Sway Config/Demo

Now, we're going to go through the Sway configuration file and explain everything 'quickly'.

The first two lines import defaults from the root Sway configuration folders.

Then, we define some variables for programs and commands to run, which we later bind to hotkeys. You can change these to whatever programs you like, if you emerge alternatives.

Below, are some command bindings for audio controls, which use `wpctl` to change the volume, and mute and unmute, the default audio input and output devices; again, these are bound to hotkeys below.

Next is the input configuration, which you already changed for your keyboard layout. If we open another terminal with `Mod + T`, you can run `man sway` to open the Sway manual, which will give more detail. `Q` to exit, and `man 5 sway` will open the manual for the Sway configuration file. It's possible to change things like mouse sensitivity with input selectors. `Mod + Q` will close the currently focused window.

Below input, is output. The first line defines the background, which is applied by the `swaybg` package we installed earlier. You could change this to an image, and set different backgrounds for different output devices if you wanted to.

Below is a commented line which explains how to change your resolution. I'm going to uncomment it, open another terminal, run `swaymsg -t get_outputs`, and set the name of the selected output to `Virtual-1`. Save, then `Mod + Shift + C` to reload my config, and my resolution changes.

Next, we set all of our keybindings. The first, is the definition for the base modifier key, then below that is the `kill` command for the currently focused window, `Mod + Shift + Q`, which opens a prompt to close Sway and log out, the configuration reload keybinding, and `Mod + L`, which will launch `swaylock` and lock our screen. We can start typing to unlock. This UI has a capslock indicator, and we can also hit `Escape` to clear the password we've typed so far.

Below are the keybindings to run certain programs. We've already used `Mod + T`, `Mod + A` opens Bemenu at the top of the screen, where we can type the name of a program. `Enter` would launch the currently highlighted program, and `Escape` quits the launcher. `Mod + W` launches our web browser, but it won't work yet as our browser isn't finished compiling. `Mod + C` opens `nano` in a new terminal, `Mod + E` opens the Ranger file explorer, where we can use the arrow keys to navigate, hit `Backspace` to show and hide hidden files, and `Space` to select multiple files.

`Mod + N` opens the `nm-tui` for network configuratoin, `Mod + M` opens `cmus` for playing music, `Mod + V` opens our volume control TUI, and `Mod + K` opens `htop` for task management.

Just like you can change any of the programs at the top, you can change the keybindings here too if you like.

If we hold `Mod`, we can use the arrow keys to move our focus between different windows, and if we hold `Mod + Shift`, we can use the arrow keys to move our windows around.

You've already used the hotkeys to switch between workspaces, but we can do `Mod + Shift`, then a number, to move the currently focused window to a different workspace.

`Mod + O` and `Mod + P` changes whether the currently focused window will split horizontally or vertically if we open a new window.

`Mod + Shift + J` switches the layout mode to stacking, `Mod + Shift + K` switches to tabs, and `Mod + Shift + L` switches back to tiling mode.

`Mod + Enter` fullscreens the currently focused window, and hitting it again exits fullscreen.

`Mod + Shift + Space` turns the currently focused window into a floating window, and `Mod + Space` switches focus between floating and tiled windows. If we hold `Mod`, left click and drag on a window, it moves. If we right click instead, we can resize it.

`Mod + Shift + Space` on a floating window puts it back into tiling mode.

We can use `Mod + Shift + Tab` to focus on the container for the currently focused window, and `Mod + Tab` focuses back on its children.

`Mod + S` takes a screenshot of the entire screen, which is placed on our clipboard, and `Mod + Shift + S` allows us to select a region to screenshot.

`Mod + Shift + Minus` moves the currently focused window to the scratchpad, and `Mod + Minus` will cycle between windows on our scratchpad. This is Sway's equivalent of window minimization.

Then we bind the audio control keys. These keybindings are for OEM, dedicated volume control keys, but you could change them to whatever you like if you don't have dedicated volume control keys.

`Mod + R` enters resize mode, which allows us to use the arrow keys to resize the currently focused window. `Enter` or `Escape` exits resize mode.

Next, we set up our `swayidle` configuration; the first line controls screen locking timeouts, the second line turns off the display after the screen is locked, and the third line sends the computer to sleep. The timeouts, 180, 10 and 600, are in seconds; you can change them to whatever you like.

Below the idle configuration, we tell Sway to use Waybar as its status bar, and anchor it to the top of the screen.

Next, we have some examples for floating window selectors; if we add one for `foot`, reload our configuration, and open `foot`, the new window defaults to floating mode.

Below that are idle inhibition selectors; we could use them to tell Sway not to lock the screen or send the computer to sleep if, for example, `MPV`, our video player, were in fullscreen mode. Many applications will handle this automatically, so this is a fallback if some application you use doesn't inhibit idle itself. You can also click the little eye in the top left of the screen; when it's orange (according to my styles), the `swayidle` will never trigger, and when it's blue, `swayidle` can do its thing.

We disable `xwayland` below that; if you want `xwayland`, which we'll look at in a future video, you'll need to add the `X` use flag to `wlroots` in your `package.use`, and make any other changes required after that, before setting `xwayland` to enabled here.

Then we tell Sway to launch Pipewire, our audio server, when we log in, and to close it when we log out.

Finally, we style the window dressings, font size, and tell Sway not to change focus if we move our mouse. You can change these styles and configuration options, along with others listed in `man 5 sway` to anything you like.

### Waybar Config

Now, lets exit the Sway configuration, and take a look at our Waybar config. There are two files: `config`, which is a `json` file which tells Waybar what modules to show, and a `style.css` file which is a `css` file which applies styles to Waybar. There's nothing that interesting here, but you can read the Waybar `man` pages, just like the Sway `man` pages, to change these configuration files however you like.

### Foot Config

Finally, there's the `foot` configuration file `foot.ini`. Not much is changed in here; I set the font size, the text cursor style, told Foot to hide the cursor while I'm typing, and set the colours and opacity for the window. You can change this file however you like, and the commented out lines cover everything from hotkeys to styling.

## Outro

That's it! We've built a basic Gentoo system with a highly configurable desktop environment, and some basic programs to get us started. I recommend you play about with your configuration files, read some more information about the packages we've installed, and the `USE` flags we've set, then click on the playlist on the screen to see what other guides I've made _so far_ for extending this environment.
