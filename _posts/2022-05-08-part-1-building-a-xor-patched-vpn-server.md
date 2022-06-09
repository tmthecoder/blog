---
title: 'XOR-Patched VPN - Part 1: The Server'
date: 2022-05-8 4:00:00 -4:00
categories: [Tutorials, XOR-patched VPN Server]
tags: [vpn, xor-patched, openvpn, obfuscated, tutorial]
author: tmthecoder
comments: true
---

# Other Parts

- [Part 2](../part-2-the-patched-desktop-clients/): The second part of this wonderful guide. Goes into an in-depth guide on building or finding the desktop clients (macOS, Linux, Windows) so you can actually connect to this VPN server from any Desktop Platform.
- Part 3: **COMING SOON!** The third part of this guide. Focuses on mobile clients. We'll be going into depth on how to build the Android client, and I'll provide information on the iOS client.

# Summary

The title is pretty self-explanatory, but I'll elaborate a little in regards to what this guide _supposed_ to teach & show. This guide will show you how to build your own self-hosted VPN server utilizing OpenVPN and the Tunnelblick XOR patch. The XOR-patched VPN adds an extra layer of obfuscation (though rudimentary) to conceal your VPN traffic from appearing as OpenVPN traffic in general packet inspection routines. There are extra layers of obfuscation that can be added on top of a server like this, and I may elaborate on those in the future as well. I personally use one of these setups since the others require another tunneling solution like stunnel.

Now, if any (or all) of those words didn't make sense, don't worry! I'm planning to explain the steps we take to build the server in depth. This guide is aimed at a beginner-level, however anyone more advanced can feel free to follow along and just skim past the more in-depth explanations I'll offer.

# Terms & Definitions

Since I said this was a beginner-level guide, here's some quick definitons & terms you'll see scattered through the post.

- OpenVPN: This is the protocol we'll be using to build our VPN server & client.
- Tunnelblick: The current maintainers of the XOR patch that we will use to add that extra layer of obfuscation to our OpenVPN server. They also made the Tunnelblick application, which you'll use to connect to the VPN server you create (if on macOS).
- XOR patch: A patch adding header scrambling through a pre-shared secret. Instead of openly carrying the OpenVPN signature, your packets will appear as encrypted traffic in basic detection setups.

# Useful Info

Here's some useful information and recommendations regarding the setup, capabailities, and limitations of what your VPN setup will be able to do.

**Please read these sections**

## Server

I recommend hosting this VPN on some cloud-based server hosting service. There are many available to pick from (AWS, Azure, Google Cloud, Oracle Cloud, etc). If you dont want to use a cloud hosted server, any Linux device will work, but you do need to open a port to connect to your VPN. We'll be using TCP port 443 in this guide, though you can host this server on any TCP or UDP port you choose.

> This guide is currently using openvpn 2.5.6. I will attempt to keep it up-to-date with the latest version that the XOR patches are written for (visible [here](https://github.com/Tunnelblick/Tunnelblick/tree/master/third_party/sources/openvpn))
{: .prompt-info }

## Client

The client side also requires a patched OpenVPN application. I will be posting instructions to create the patched clients if they aren't openly available. Every client outside of macOS will require a fair bit of technical knowledge. I will make these into separate guides to unnecessarily lengthening this guide. Here's a brief description of the platform-specific statuses:

### iOS

There currently exists no open-soruce and openly available XOR-patched VPN cliens. Sadly, there aren't clients that use the same OpenVPN source that we can patch like Android does. If I'm able to find (or make) a solution regarding a patched iOS client, I'll update this section.

### Android

We will create a client using the same XOR patch on the [OpenVPN for Android](https://github.com/schwabe/ics-openvpn) application. While there may be already built APKs, I'd refrain from using them since you can't know for sure that the patch was the _only_ code addition to the final APK. Plus, you get to go through the experience of building the Android app on your own, which just adds a little special touch on top of already using a custom-built server.

### macOS

This is the only platform with a prebuilt application. As I mentioned before, Tunnelblick maintains the XOR patch and also includes it in all builds of their VPN client. [Tunnelblick](https://tunnelblick.net) is open-source and provides an amazing VPN client for macOS devices. Your best bet is to download and use Tunnelblick on macOS (Hint: I use it).

### Windows

We'll also build our own Windows client, though this guide may be slightly shaky since I barely use Windows. You'll need Linux to cross-compile this client though, I do not know if there's a way to actually build it on Windows itself and I don't plan on trying since I'd have to do it on a VM (which is slow and painful).

### Linux

Aside from macOS, this is the easiest platform to use with XOR OpenVPN servers. We'll be applying a subset of the same steps for the patched client as we did with building the server, so it's a relatively easy task.

# Getting Started

Now that we've got all of the introductory info out of the way, it's time to get started. We have three main steps to build this server:

1. Download everything
2. Patch & Build our OpenVPN Server
3. Setup Certificates with EasyRSA
4. Generate Client & Server OpenVPN configurations

So, let's get started!

# 1. Downloads

## Dependencies

The dependencies are pretty standard, just the tools to download the source and build it. You may even have most of these installed if you've used that device for builds.

Run the command below to install the needed dependencies

```terminal
sudo apt install git gcc libssl-dev liblzo2-dev liblz4-dev libpam0g-dev make unzip
```
> If your server isn't on Ubuntu Server, you may need to install additional/other dependencies using your native package manager
{: .prompt-warning}

## EasyRSA Source

We'll download & unzip the posted archive of easy-rsa from their GitHub:

```terminal
wget -O easy-rsa-src.zip https://github.com/OpenVPN/easy-rsa/archive/master.zip

unzip easy-rsa-src.zip
```
## OpenVPN Source

We'll download the OpenVPN source code by downloading the zipped version compatible with the XOR patch. Since the latest XOR patch version is 2.5.6, we'll download & unzip the respective ZIP file from OpenVPNs server.

```terminal
wget https://swupdate.openvpn.org/community/releases/openvpn-2.5.6.zip

unzip openvpn-2.5.6.zip
```

## OpenVPN Init File

Since we want the VPN to auto start when we turn our server on, we're going to download a quick init script that lets us do this:

```terminal
wget -O initd-openvpn https://gist.githubusercontent.com/john564/6765292/raw/c5f8e1be80c7cc8c94f3c287f887ee79f2d028e9/etc%2520init.d%2520openvpn
```
> This script is meant for Debian-based Linux distributions. If your linux distribution isn't Debian-based, you may need to find the appropriate one for your Linux distribution
{: .prompt-warning}

## XOR Patch Files

We're going to download each of the 6 Tunnelblick XOR patch files hosted on their GitHub. For organization sake, we'll make a directory in the openvpn source directory to store the patches:

```terminal
mkdir openvpn-2.5.6/patches
```

Use these commands to download each patch:

```terminal
wget -O openvpn-2.5.6/patches/xorpatch_1.diff https://raw.githubusercontent.com/Tunnelblick/Tunnelblick/master/third_party/sources/openvpn/openvpn-2.5.6/patches/02-tunnelblick-openvpn_xorpatch-a.diff

wget -O openvpn-2.5.6/patches/xorpatch_2.diff https://raw.githubusercontent.com/Tunnelblick/Tunnelblick/master/third_party/sources/openvpn/openvpn-2.5.6/patches/03-tunnelblick-openvpn_xorpatch-b.diff

wget -O openvpn-2.5.6/patches/xorpatch_3.diff https://raw.githubusercontent.com/Tunnelblick/Tunnelblick/master/third_party/sources/openvpn/openvpn-2.5.6/patches/04-tunnelblick-openvpn_xorpatch-c.diff

wget -O openvpn-2.5.6/patches/xorpatch_4.diff https://raw.githubusercontent.com/Tunnelblick/Tunnelblick/master/third_party/sources/openvpn/openvpn-2.5.6/patches/05-tunnelblick-openvpn_xorpatch-d.diff

wget -O openvpn-2.5.6/patches/xorpatch_5.diff https://raw.githubusercontent.com/Tunnelblick/Tunnelblick/master/third_party/sources/openvpn/openvpn-2.5.6/patches/06-tunnelblick-openvpn_xorpatch-e.diff
```
> We're renaming the files to make them easier to apply later on. There's no need to do this aside from personal convenience.
{: .prompt-info }

# 2. OpenVPN

Now that we've downloaded everything, we're going to patch and build our server.

Patching & building is pretty straightforward, all we do is apply eatch patch in our source directory, then build the openvpn server through their build guide.

## Applying the Patches

To apply the patches, use the following commands:

```terminal
cd ~/openvpn-2.5.6

git apply patches/xorpatch_1.diff

git apply patches/xorpatch_2.diff

git apply patches/xorpatch_3.diff

git apply patches/xorpatch_4.diff

git apply patches/xorpatch_5.diff
```
> We're applying the patches we downloaded before. Now I'm guessing you can tell why I renamed them. You can just change the number between each command easily.
{: .prompt-info}

## Building the Server

Now that our patches are applied, we can build the server like OpenVPN's official guide mentions.

Building and installing is relatively straightforward, and just needs 3 commands:

```terminal
./configure --prefix=/usr

make

sudo make install
```

The first command configures our build environment, setting us up to run `make`, which will compile our OpenVPN server. `sudo make install` will install the server and allow us to use the `openvpn` command in our terminal.

Now that we have openvpn all set up, we'll start creating our user-specific certificates using EasyRSA

> These certificates will be used in our VPN configuration files in step 4
{: .prompt-info}

# 3. EasyRSA

We're going to use EasyRSA to create our VPN keys and certificates. First, we'll start with making a few configuration modifications to what we want to do.

This server will use ECC (Elliptic Curve Cryptography) for key & certificate generation.

## Setting our vars

First, we have to remove the example vars file and create our own to enable ECC:

```shell
rm ~/easy-rsa-master/easyrsa3/vars.example

touch ~/easy-rsa-master/easyrsa3/vars.example
```
> For some reason, pasting our lines below into the `vars` file instead of `vars.example` doesn't use the right config file when we move onto further steps
{: .prompt-info}

Now, we can paste in the following lines into `~/easy-rsa-master/easyrsa3/vars.example`{: .filepath}

```shell
set_var EASYRSA_ALGO ec

set_var EASYRSA_CURVE secp521r1
```
> Do this with any text editor of your choice. If you're new to command-line editing, `nano` is an easy way to start, though I definitely recommend picking up `vim` if you're interested.
{: .prompt-info}

## Creating Keys & Certificates

Now that we've setup the EasyRSA vars, we can start generating the necessary keys & certificates. We'll start with the server-side stuff. Enter the following commands to create our server's `ta.key`, initialize EasyRSA's PKI, and create our server's configuration information.

```terminal

cd ~/easy-rsa-master/easyrsa3/

openvpn --genkey tls-crypt-v2-server tls-v2.key

./easyrsa init-pki

./easyrsa --batch build-ca nopass

./easyrsa --batch build-server-full server nopass
```

Now it's time to generate information for our clients. Enter the following command to generate a key & certificate for a client:

```terminal
./easyrsa --batch build-client-full [client-name] nopass

openvpn --genkey tls-crypt-v2-client pki/issued/[client-name].tls-v2.key --tls-crypt-v2 tls-v2.key
```

> These commands need to be repeated for each client (device) that will use this server. Many other commands will have this same repitition.
{: .prompt-warning}

# 4. Configuration

Now that we've got our certificates all created, it's time to actually configure our server & client profiles.

## Server

We're going to start with the server. First, we'll create a boilerplate `server.conf` configuration. I'm going to do this in the `~/easy-rsa-master/easyrsa3/pki/server`{: .filepath} directory:

```terminal
mkdir ~/easy-rsa-master/easyrsa3/pki/server

cd ~/easy-rsa-master/easyrsa3/pki/server

nano server.conf
```

In the editor window, paste the following boilerplate config (I'll explain it in a bit):

```shell
port 443
proto tcp
scramble obfuscate --A-Very-Secure-Random-Password--
dev tun
server 10.8.0.0 255.255.255.0
cipher AES-256-GCM
compress
persist-key
persist-tun
user nobody
group nogroup
status /etc/openvpn/openvpn-status.log
verb 3
push "redirect-gateway def1"
push "dhcp-option DNS 1.1.1.1"
push "dhcp-option DNS 1.0.0.1"
keepalive 5 30
dh none

<ca>
# CONTENTS OF ~/easy-rsa-master/easyrsa3/pki/ca.crt HERE
</ca>

<cert>
# CONTENTS OF ~/easy-rsa-master/easyrsa3/pki/issued/server.crt HERE
</cert>

<key>
# CONTENTS OF ~/easy-rsa-master/easyrsa3/pki/private/server.key HERE
</key>

<tls-crypt-v2>
# CONTENTS OF ~/easy-rsa-master/easyrsa3/tls-v2.key HERE
</tls-crypt-v2>
```
Okay, so that's a good bit of stuff to look at. Let's start at the top:

- `port 443` & `proto tcp`: We're telling openvpn to use TCP port 443 for incoming VPN connections. This is the port HTTPS traffic goes on, so it's guranteed to be open on almost any network

- `scramble obfuscate --A-Very-Secure-Random-Password--`: This is what the XOR-patch is all about. We're scrambling our header with the password after `obfuscate`. I usually make this a base64-encoded random string

- `dev tun`: Uses a `tun` device. This one's a little in depth if I explain, but pretty much it's telling OpenVPN what virtual network device to use

- `server 10.8.0.0 255.255.255.0`: Sets the IP range for the server & its clients. When clients connect to the vpn they'll be given an ip that looks like `10.8.0.x` and the server will always be `10.8.0.1`

- `cipher AES-256-GCM`: Use the AES-GCM cipher for client-to-server encryption

- `compress`: Compress traffic. Usually helps performance

- `persist-key` and `persist-tun`: Doesn't reopen/reread the key & tun devices across restarts

- `user nobody` and `group nogroup`: Sets the user & group of the executable to none

- `status /etc/openvpn/openvpn-status.log` & `verb 3`: Sets the output file of the status log and the verbosity that the logging should follow. Verbosity ranges from 1-11 where 11 is the most verbose

- `push "redirect-gateway def1"`: Tells the client to edirect all traffic through the VPN

- `push "dhcp-option DNS x.x.x.x"`: Tells the client to use the given DNS servers. I like to use Cloudflare's `1.1.1.1`, but you can change this as you need

- `keepalive 5 30`: Keeps the connection alive in times of inactivity to avoid losing the VPN connection

- `dh none`: We aren't using a dh key since it isn't needed with ECC-based servers

- Each of the `<ca></ca>`, `<cert></cert>`, etc. are all keys & certificates we made in previous steps. More info on those follows.

Okay, so there's our explanation of each of the config options we set. There are a lot more if you want to dive into them [here](https://openvpn.net/community-resources/reference-manual-for-openvpn-2-4/)

Now, just two steps remain for the server configuration: Creating a password for the `scramble` option and filling in our certificate information.
- To create the password, I recommend using any random password generator and copying the string where it says `--A-Very-Secure-Random-Password--`

- To inline our certificates, copy the text contents of each file in the comment. You can do this by running `cat [filepath]` and copying the optput between (and including) `-----BEGIN` and `-----END` into the section between the two angled brackets with the same word. Below is a simple example of how it should look

```console
<ca>
-----BEGIN CERTIFICATE-----
# Certificiate contents...
-----END CERTIFICATE-----
</ca>
```

Once you're done with that, we need to copy the configration into OpenVPN's default server directory:

```terminal
sudo mkdir /etc/openvpn

sudo cp server.conf /etc/openvpn/server.conf
```

Now we just need to copy our init script from the start and register OpenVPN's service to start on boot:

```terminal
sudo cp ~/initd-openvpn /etc/init.d/openvpn

sudo chmod a+x /etc/init.d/openvpn

sudo update-rc.d openvpn defaults
```
> Since the init script is distro-specific, the `update-rc.d` command may not work depending on your linux distro. If it doesn't you'll need to see how to add startup services on your linux distribution.
{: .prompt-warning}


Congratulations! You just set up an XOR-patched OpenVPN Server! You built OpenVPN from source, generated custom server certificates, and configured a custom VPN server!!

## Client

You just set up the server, but there's still one step left before you can actually connect to it: Creating client profiles (duh).

This is actually extremely similar to setting up the server configuration, just with a few config variations. We're going to do this setup a little differently, since you might be setting up a ton of clients.

Since I'm extremely thoughtful, I've made a quick script that does a lot of the manual work for you in terms of certificate copying. The script is below and is explained in the comments.

```shell

#!/bin/bash

# First argument is the template file's path
template_name=$1

# Second argument is the client name (no extension since keys/certs are all named with the client name)
client_name=$2

# We generate the output file path for later use
output_file="$HOME/easy-rsa-master/easyrsa3/pki/client/$client_name.ovpn"

# Make the output directory if it doesn't exist
if [ ! -d ~/easy-rsa-master/easyrsa3/pki/client ]; then
    mkdir -p ~/easy-rsa-master/easyrsa3/pki/client
fi

# Delete the output file if it's already made
if [ -f "$output_file" ]; then
    rm "$output_file"
fi

# Put the contents of the template into the ouput file
cat "$template_name" >> "$output_file"


# Put each of the certificates & keys into the client output file
{
    # Puts the generic ca.crt into the output file
    echo "<ca>";
    sed -n '/-----/,$p' ~/easy-rsa-master/easyrsa3/pki/ca.crt;
    echo "</ca>";

    # Puts the contents of the client's cert file into the output file
    echo "<cert>";
    sed -n '/-----/,$p' "$HOME/easy-rsa-master/easyrsa3/pki/issued/$client_name.crt";
    echo "</cert>";

    # Puts the contents of the client's key file into the output file
    echo "<key>";
    sed -n '/-----/,$p' "$HOME/easy-rsa-master/easyrsa3/pki/private/$client_name.key";
    echo "</key>";

    # Puts the contents of the client's tls-crypt-v2 key into the output file
    echo "<tls-crypt-v2>";
    sed -n '/-----/,$p' "$HOME/easy-rsa-master/easyrsa3/pki/issued/$client_name.tls-v2.key" >> "$output_file";
    echo "</tls-crypt-v2>";
} >> "$output_file"

```

Paste the above contents into a script located at `~/easy-rsa-master/easyrsa3/pki/client/client-gen.sh`{: .filepath} (You'll need to run `mkdir ~/easy-rsa-master/easyrsa3/pki/client` first)

We'll set the script as executable:

```terminal
cd ~/easy-rsa-master/easyrsa3/pki/client

chmod a+x client-gen.sh
```

Okay, we're almost done! Now, we just need the template file and we can create our client configurations!

For the template file, paste the following into `~/easy-rsa-master/easyrsa3/pki/client/template.ovpn`{: .filepath}

```shell
client
dev tun
proto tcp
scramble obfuscate --Your-Very-Secure-Password-From-Before--
remote --Your-Server-IP-Address-- 443
resolv-retry infinite
nobind
persist-key
persist-tun
remote-cert-tls server
cipher AES-256-GCM
compress
verb 3
```

Before we get to calling the script, I'll give a brief overview of the client template:
- `client`: Tells openvpn that this is a client config (shocker)

- `proto tcp` & `scramble obfuscate --Your-Very-Secure-Password-From-Before--`: These lines should match word-for-word between server and client configs. Same meanings on both sides. Make sure `--Your-Very-Secure-Password-From-Before--` is the same password as on the server config

- `remote --Your-Server-IP-Address-- 443`: Tells openvpn where to connect. Replace `--Your-Server-IP-Address--` with the Public IP address of your server

- `resolv-retry infinite`: Automatically retry if a connection can't be made

- `nobind`: Don't bind to a local address & port. It will use a dynamic port on the client side

- `persist-key` & `persist-tun`: Same functionality as server config

- `remote-cert-tls server`: Require that the server certificate is signed

- `cipher AES-256-GCM` & `compress`: Same functionality as server config. Must match server config

Okay, now it's time to make each of our client configs.

To do this, run the following command for each client you ran the previous easyrsa commands for. The names must also match the client names you used then.

```terminal
./client-gen.sh template.ovpn [client-name]
```
> Run this once per client. If you don't remember the names you used run `ls ~/easy-rsa-master/easyrsa3/pki/issued` and you'll see all of the `.crt` files available. Run it once for each `.crt` file that isn't `server.crt`
{: .prompt-info}

Once you've run the command, you'll see the `~/easy-rsa-master/easyrsa3/pki/client`{: .filepath} folder contain `.ovpn` files for each name you entered. These are the files you'll move to each device you want to connect to the VPN from.

Now, you're all done with OpenVPN! There's just a few technical details with IPs and Ports that we need to finish up before your server will be all set to accept VPN connections!

## IP Forwarding & Ports

We just need to do 3 things to prepare our server for VPN traffic.

Frist, we need to enable IPV4 forwarding. To do this, we want to edit `/etc/sysctl.conf`{: .filepath}
```terminal
sudo nano /etc/sysctl.conf
```
> I'm not sure if this file differs between linux distros. If it does, you'll need to see what your distro's equivalent is.
{: .prompt-warning}

In the file, look for a line that reads `#net.ipv4.ip_forward=1`. All you need to do is remove the `#` from the line so it reads: `net.ipv4.ip_forward=1`.

Next, we need to allow the traffic to pass through the local firewall via `iptables`. To do this, we'll use another script so these rules can be easily reset if they are cleared.

Copy the following script into `~/firewall-config.sh`{: .filepath}:

```shell
#!/bin/bash
iptables -t filter -F
iptables -t nat -F
iptables -A FORWARD -m state --state RELATED,ESTABLISHED -j ACCEPT
iptables -A FORWARD -s "10.8.0.0/24" -j ACCEPT
iptables -t nat -A POSTROUTING -s "10.8.0.0/24" -j MASQUERADE
```

And make it executable:

```terminal
chmod a+x ~/firewall-config.sh
```

This script just tells the firewall to accept and route traffic from our VPN IP range through the server.

Now we just have one remaining step: port forwarding.

You need to forward the port that we configured on our VPN server & client configs (this guide used TCP port 443). I won't go into forwarding a port here since this has to be done on either your network provider's portal (if you're at home) or the cloud provider's portal (if using a cloud instance), all of which differ based on the provider. Once you forward the port, you'll be able to connect from any of the clients we mentioned above.

Once that's done, reboot the server and run our firewall script

```shell
sudo reboot now

sudo ~/firewall-config.sh
```

# 5. Going Forward

## Desktop Connections

Connecting on Desktop Clients is relatively simple (except Windows). Like I mentioned at the beginning, macOS has [Tunnelblick](https://tunnelblick.net) and Linux can just use the same process to patch OpenVPN. I'll make a separate guide for Linux soon, but if you're feeling ambitious, try to patch OpenVPN the same way we did to the `sudo make install` step and then use `sudo openvpn profile.ovpn` to connect to the VPN.

I'll be making a more in-depth guide for Windows soon as well as we need to compile a `.exe` file on Linux using a similar technique, but with some more in-depth steps.

## Mobile connections

Both mobile platforms have no XOR patch supporting clients on their respective app stores, so we have to compile an app by ourselves. I'm still looking into how to do this for iOS, but know how to for Android.

I'll make a guide for Android once the Desktop guides are created. In summary, we have to patch the OpenVPN C source the same way we did here, but within the contexts of the [OpenVPN for Android](https://github.com/schwabe/ics-openvpn) project.

For iOS, I'm looking into a method to patch [Passepartout VPN](https://passepartoutvpn.app), but I'd have to write my own XOR patch for it, so it may take a while. I'll update this guide if (or when) a solution is available.

# Last Remarks

Congratulations! You built your own XOR-patched OpenVPN server! There is a discussion available below if you have any questions, so feel free to ask there or email me at [tmthecoder@gmail.com](mailto:tmthecoder@gmail.com). Thanks for reading and I hope you learned something!
