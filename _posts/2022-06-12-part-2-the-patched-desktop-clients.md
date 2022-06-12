---
title: 'XOR-Patched VPN - Part 2: The Desktop Clients'
date: 2022-06-12 27:00:00 -4:00
categories: [Tutorials, XOR-patched VPN Server]
tags: [vpn, xor-patched, openvpn, obfuscated, tutorial]
author: tmthecoder
comments: true
---

# Other Parts

- [Part 1](../part-1-building-a-xor-patched-vpn-server/): A guide on building the actual VPN server. This explains a lot of the terminology too, so make sure to give it a read if you haven't since this guide will definitely build off of it in some areas.
- Part 3: **COMING SOON!** The third part of this guide. Focuses on mobile clients. We'll be going into depth on how to build the Android client, and I'll provide information on the iOS client.

# Summary

Welcome to the second part of my three-part XOR-Patched VPN Guide. This guide will focus on building or donwloading clients for each of the three Desktop platforms (macOS, Linux, and Windows). I'll start with macOS since it's the simplest. We'll use [Tunnelblick](https://tunnelblick.net) since it already has support for XOR-patched VPN servers. Linux is next since it's of relatively little difficulty since you've already done most of the work for it before (the process is almost exactly the same as setting up the server). Last is going to be windows since it's easily the most time-consuming and slightly in-depth, though not too much.

This guide will be like the last one, but sectioned a little differently so you only follow what you need to. The headings will be per-platform, so scroll to the one you need and follow from there!

# Platform Agnostic Setup

Before I dive into the specifics of each platform's OpenVPN client setup, there's one step that needs to be taken regardless of the client's platform: moving the `.ovpn` file from your OpenVPN server to the client machine. Many of you may know how to do this already, but I'll include a quick piece on how to do this if you haven't done it before or may not know/remember how to.

We're going to use the `scp` command to do this. I'm hoping you used `ssh` to execute the commands on the last turorial since that would speed up your time going from the guide to executing the terminal commands or filling in the scripts _a lot_. `scp` uses the same underlying port as SSH, but is instead made for copying files between machines.

All you need to do is type in `scp username@server-address:path/to/remote/file path/to/local/file`. So, if I wanted to transfer the file named `tejas.ovpn`{: .filepatg} to my local computer at `Downloads/tejas.ovpn`{: .filepath}, I'd type in the following:

```terminal
scp my-username@my-server-address:easy-rsa-master/easyrsa3/pki/client/tejas.ovpn Downloads/tejas.ovpn
```
> Remember that `my-username` and `my-server-address` are the username and server address that you used when (hopefully) SSH'ing into the machine for the last guide's contents. The `easy-rsa-master/...` path was where we made the configurations last time.
{: .prompt-info}

Now that that's all done, let's get started on actually making these profiles useful!

# macOS

Like I said, macOS will be extremely simple to set up. Just head over to [Tunnelblick's website](https://tunnelblick.net) and click the 'Downloads' tab. Then click either the 'Beta' or 'Stable' versions and let them download. Both versions are usually fine to download. Beta downloads get quicker releases with slight bugs sometimes. Stable releases are usually very well tested and contain almost no bugs or issues. If you're not _super_ technical, just click stable and let it download.

Once the download finishes, open the downloaded `.dmg` file and go through the guided install. You'll need to provide a password a few times since there are some secure operations being done. I do trust the application in its entirety (it's [open source](https://github.com/Tunnelblick/Tunnelblick)) just to provide some confidence to its asking for your password.

After that, the application icon will show up on your menu bar (top bar). Click on it and press the 'VPN Details...' button. That should open up a pane with an empty sidebar.

All you need to do is drag the `.ovpn` config file you downloaded earlier to this sidebar. Once you do that, the app will prompt you to install the configuration. Go through the steps as you see fit and provide your password when it asks. Once you go through those steps, the file name you selected should appear in the sidebar that was blank before and on the manu after you click the application's icon on the top bar. To connect, just click the Tunnelblick icon and press 'Connect [File Name]'. That's it, you've set up a macOS client to connect to your own XOR-Patched VPN Server!

> Your password is requested during the process to copy the configuration file's contents to a secure location. This is done to make sure your configuration isn't tampered with. It will also ask for a password if you change or rename the configuration through Tunnelblick's options.
{: .prompt-info}

# Linux

Linux is not _as easy_ as macOS, but it's __extremely__ similar to the first part of this guide where we build the server. Like, I mean word-for-word/step-by-step similar except we don't need to bother with any of the EasyRSA or Configuration setup. So, just like the last guide, I'll split this into a few steps:

1. Download everyrhing
2. Patch & Build the OpenVPN Client
3. Connect to the VPN

## 1. Downloads

We're gonna download the same dependencies as before:

```terminal
sudo apt install git gcc libssl-dev liblzo2-dev liblz4-dev libpam0g-dev make unzip
```
> If your Linux Machine isn't on Ubuntu, you may need to install additional/other dependencies using your native package manager
{: .prompt-warning}

And now the same OpenVPN Source:


```terminal
wget https://swupdate.openvpn.org/community/releases/openvpn-2.5.6.zip

unzip openvpn-2.5.6.zip
```

And the XOR same patch files:


```terminal
mkdir openvpn-2.5.6/patches

wget -O openvpn-2.5.6/patches/xorpatch_1.diff https://raw.githubusercontent.com/Tunnelblick/Tunnelblick/master/third_party/sources/openvpn/openvpn-2.5.6/patches/02-tunnelblick-openvpn_xorpatch-a.diff

wget -O openvpn-2.5.6/patches/xorpatch_2.diff https://raw.githubusercontent.com/Tunnelblick/Tunnelblick/master/third_party/sources/openvpn/openvpn-2.5.6/patches/03-tunnelblick-openvpn_xorpatch-b.diff

wget -O openvpn-2.5.6/patches/xorpatch_3.diff https://raw.githubusercontent.com/Tunnelblick/Tunnelblick/master/third_party/sources/openvpn/openvpn-2.5.6/patches/04-tunnelblick-openvpn_xorpatch-c.diff

wget -O openvpn-2.5.6/patches/xorpatch_4.diff https://raw.githubusercontent.com/Tunnelblick/Tunnelblick/master/third_party/sources/openvpn/openvpn-2.5.6/patches/05-tunnelblick-openvpn_xorpatch-d.diff

wget -O openvpn-2.5.6/patches/xorpatch_5.diff https://raw.githubusercontent.com/Tunnelblick/Tunnelblick/master/third_party/sources/openvpn/openvpn-2.5.6/patches/06-tunnelblick-openvpn_xorpatch-e.diff
```
> We're renaming the files to make them easier to apply later on. There's no need to do this aside from personal convenience.
{: .prompt-info }

## 2. Building OpenVPN

We are quite literally going to build this OpenVPN client the __exact same__ way we built the OpenVPN server before.

First, we'll apply the patches by using the following commands:

```terminal
cd ~/openvpn-2.5.6

git apply patches/xorpatch_1.diff

git apply patches/xorpatch_2.diff

git apply patches/xorpatch_3.diff

git apply patches/xorpatch_4.diff

git apply patches/xorpatch_5.diff
```

Now, we follow the exact same instructions as before to build OpenVPN:

```terminal
./configure --prefix=/usr

make

sudo make install
```

And that's it! No need to do anything else! The `openvpn` command will be available, so all you have to do is connect to your server.

## 3. Connecting

Now that the Linux machine has the `openvpn` command available, you just type in the following to connect to the OpenVPN server:

```terminal
sudo openvpn [path/to/profile.ovpn]
```

This will connect to the OpenVPN server, and that's all you have to do! Congratulations, you've set up a Linux machine to connect to your own custom-built OpenVPN server!

# Windows

The last of the 3 Desktop platforms---and the toughest---is Windows. Okay, no it's not really _that tough_, but I just tend to complain because it's a little more involved than the other two.

We'll use the [openvpn-build](https://github.com/OpenVPN/openvpn-build) project to handle the actual building for us. We will build the executable on Linux and you'll be responsible for transferring that built file the same way you did the `.ovpn` profile earlier.

Just like the Linux guide, I'll split it into a few sectioned steps:

1. Download everyrhing
2. Patch & Build the OpenVPN Client
3. Connect to the VPN

So, let's get started!

## 1. Downloads

We'll start off by downloading a few needed dependencies for the build:

```terminal
sudo apt install mingw-w64 git man2html dos2unix nsis osslsigncode unzip make gcc libssl-dev liblzo2-dev liblz4-dev libpam0g-dev autoconf libtool docutils-common
```
> If your Linux Machine isn't on Ubuntu, you may need to install additional/other dependencies using your native package manager
{: .prompt-warning}

Now we'll clone the [openvpn-build](https://github.com/OpenVPN/openvpn-build) project:

```terminal
git clone https://github.com/OpenVPN/openvpn-build
```

And download all of the patches:

```terminal
wget -O openvpn-build/generic/patches/openvpn-2.5.6-xorpatch_1.patch https://raw.githubusercontent.com/Tunnelblick/Tunnelblick/master/third_party/sources/openvpn/openvpn-2.5.6/patches/02-tunnelblick-openvpn_xorpatch-a.diff

wget -O openvpn-build/generic/patches/openvpn-2.5.6-xorpatch_2.patch https://raw.githubusercontent.com/Tunnelblick/Tunnelblick/master/third_party/sources/openvpn/openvpn-2.5.6/patches/03-tunnelblick-openvpn_xorpatch-b.diff

wget -O openvpn-build/generic/patches/openvpn-2.5.6-xorpatch_3.patch https://raw.githubusercontent.com/Tunnelblick/Tunnelblick/master/third_party/sources/openvpn/openvpn-2.5.6/patches/04-tunnelblick-openvpn_xorpatch-c.diff

wget -O openvpn-build/generic/patches/openvpn-2.5.6-xorpatch_4.patch https://raw.githubusercontent.com/Tunnelblick/Tunnelblick/master/third_party/sources/openvpn/openvpn-2.5.6/patches/05-tunnelblick-openvpn_xorpatch-d.diff

wget -O openvpn-build/generic/patches/openvpn-2.5.6-xorpatch_5.patch https://raw.githubusercontent.com/Tunnelblick/Tunnelblick/master/third_party/sources/openvpn/openvpn-2.5.6/patches/06-tunnelblick-openvpn_xorpatch-e.diff
```
> This renaming step is actually important and is __NEEDED__. If you don't rename the patches with `openvpn-2.5.6-` prefiexed to their name and `.patch` as their file extension, they will not be applied by the build script.
{: .prompt-warning}

## 2. Building the OpenVPN Executable

Now that you have all of the sources, it's time to build the OpenVPN Windows executable. Before we get started on the actual build command, we're going to make sure the OpenVPN version that will be downloaded by the build script matches that of the server and the patches.

To do this, open up the file `openvpn-build/generic/build.vars`{: .filepath} in a text editor of your choosing.

Scroll to the line that looks like

```shell
OPENVPN_VERSION="${OPENVPN_VERSION:-2.5.4}"
```
> Yours may not say `2.5.4` at the end, but the rest of the line should look the same
{: .prompt-info}

Change the `2.5.4` or whatever number you have to the same number as our patch. In this case, mine would be changed to:

```shell
OPENVPN_VERSION="${OPENVPN_VERSION:-2.5.6}"
```

Now, we're all set to build OpenVPN.

To do this, we'll go into the `windows-nsis` directory and run the shell script:

```terminal
cd openvpn-build/windows-nsis/

./build-complete
```

Once it finishes, you should have a file in that same directory named `openvpn-install-[version]-[arch].exe`{: .filepath} or some file that looks like that and ends in `.exe`. That's your patched Windows installer!

## 3. Connecting to the VPN

Now that you have that build `.exe` file, it's time to transfer it to your Windows machine.

Once you do that, double click it (if a warning prompt comes up, you can safely ignore/proceed since you built the package), and go through the installer.

Once it finishes, you can delete that `.exe` file as another will show up, which is the actual OpenVPN client. Open that application and use the UI to import the `.ovpn` profile you transferred from the VPN server earlier.

You can then connect and everything _should_ work and you'll be browsing the web through your custom-made VPN on Windows!

# Final Remarks

Congratulations! You've made the VPN profiles that we made in the last guide useful by configuring your own XOR-patched OpenVPN clients! There is a discussion available below if you have any questions, so feel free to ask there or email me at [tmthecoder@gmail.com](mailto:tmthecoder@gmail.com). Thanks for reading and I hope you learned something!
