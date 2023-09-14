+++
categories = ["Linux"]
date = "2012-09-20T07:23:00Z"
description = "CentOS/RHEL: Installing SHOUTcast Server"
tags = ["shoutcast"]
title = "CentOS/RHEL: Installing SHOUTcast Server"
slug = "centosrhel-installing-shoutcast-server"
+++

**Note:** This article is outdated now.

#### Introduction:

This guide will explain you how to install SHOUTcast DNAS 2.0 server on CentOS/RHEL. SHOUTcast lets you stream live music to listeners and start your own radio station on the web. For more information and a full list of features please visit their [website](http://www.shoutcast.com/BroadcastNow "SHOUTcast").

#### What is SHOUTcast?

The SHOUTcast Distributed Network Audio Server v2 (SHOUTcast DNAS 2.0) is the next generation in SHOUTcast broadcasting server technology. Designed to work with the new SHOUTcast 2.0 YP platform.

**Warning:** Please be aware of copyright laws. You cannot stream music unless you own the royalties to it or have permission from the owner, I'm NOT responsible for what you use this tutorial to do, stream and set up your own server at your own risk.

I also assume you have some Linux knowledge.

#### Installing SHOUTcast DNAS

The SHOUTcast Distributed Network Audio Server (DNAS 2.0) is responsible for the actual streaming and broadcasting of your audio content out to the world.

**Note:** If you are using some firewall don't forget to make exception on the appropriate ports.

**Attention:** Please DO NOT run the DNAS as a root for the security reason, instead we'll create a shoutcast user

    adduser shoutcast

Update the password of 'shoutcast' using

    passwd shoutcast

Now login as the new shoutcast user, or you can su to the user

    su - shoutcast

Create a directory shoutcast

    mkdir shoutcast
    cd shoutcast

First we need to download SHOUTcast server from nullsoft, (we are using 64-bit system)

    wget http://download.nullsoft.com/shoutcast/tools/sc_serv2_linux_x64_07_31_2011.tar.gz

We also need SHOUTcast Transcoder

**Note:** If you are planning to [stream music from Winamp](#installing-winamp-on-windows-and-streaming-to-a-shoutcast-server) you can skip the section about SHOUTcast transcoder.

**Warning:** "In order to broadcast in MP3 format you will need to purchase an MP3 encoding license."

    wget http://download.nullsoft.com/shoutcast/tools/sc_trans_linux_x64_10_07_2011.tar.gz

Extract both files:

    tar xzf sc_serv2_linux_x64_07_31_2011.tar.gz ; tar xzf sc_trans_linux_x64_10_07_2011.tar.gz

Shoutcast has now been installed and you can tidy up the directory

    rm -f sc_serv2_linux_x64_07_31_2011.tar.gz sc_trans_linux_x64_10_07_2011.tar.gz

You should have a similar list of files and directories as below

```
[shoutcast@test shoutcast]$ ls -l
total 6652
drwxrwxr-x 2 shoutcast shoutcast    4096 Oct  7 21:41 calendar
-rwxrwxr-x 1 shoutcast shoutcast    8331 Oct  7 19:03 changes.txt
drwxrwxr-x 5 shoutcast shoutcast    4096 Oct  7 21:41 config_builder
drwxrwxr-x 2 shoutcast shoutcast    4096 Jul 31 17:51 control
drwxrwxr-x 3 shoutcast shoutcast    4096 Oct  7 21:41 docs
drwxrwxr-x 2 shoutcast shoutcast    4096 Oct  7 21:41 logs
drwxrwxr-x 2 shoutcast shoutcast    4096 Oct  7 21:41 music
drwxrwxr-x 3 shoutcast shoutcast    4096 Oct  7 21:41 playlists
-rwxrwxr-x 1 shoutcast shoutcast    2896 Jul 28 17:38 readme.txt
-rwxrwxr-x 1 shoutcast shoutcast 1901504 Jul 31 17:25 sc_serv
-rwxrwxr-x 1 shoutcast shoutcast    1539 Jun 18  2011 sc_serv_basic.conf
-rwxrwxr-x 1 shoutcast shoutcast     898 Jun 18  2011 sc_serv_debug.conf
-rwxrwxr-x 1 shoutcast shoutcast     990 Jun 18  2011 sc_serv_public.conf
-rwxrwxr-x 1 shoutcast shoutcast    1066 Jun 18  2011 sc_serv_relay.conf
-rwxrwxr-x 1 shoutcast shoutcast    1317 Jun 18  2011 sc_serv_simple.conf
-rwxrwxr-x 1 shoutcast shoutcast 4765076 Oct  7 19:46 sc_trans
-rwxrwxr-x 1 shoutcast shoutcast    2487 Oct  5 19:11 sc_trans_basic.conf
-rwxrwxr-x 1 shoutcast shoutcast     621 Jun 18  2011 sc_trans_capture.conf
-rwxrwxr-x 1 shoutcast shoutcast     895 Jun 18  2011 sc_trans_debug.conf
-rwxrwxr-x 1 shoutcast shoutcast     942 Jul  7 18:15 sc_trans_dj.conf
-rwxrwxr-x 1 shoutcast shoutcast     909 Jul  7 18:16 sc_trans_playlist.conf
-rwxrwxr-x 1 shoutcast shoutcast    1442 Jul  7 18:25 sc_trans_simple.conf
-rwxrwxr-x 1 shoutcast shoutcast   28442 Jan 28  2011 tos.txt
drwxrwxr-x 2 shoutcast shoutcast    4096 Oct  7 21:41 vuimages
```

As you can see there is a few sample configuration files for SHOUTcast server and transcoder. We are not going to explain each options as they are well documented in the configuration files itself. There is also documentation directory (docs) where you can find all necessary informations.

Alternatively you can use web based configuration tool located in "config_builder" directory. If you have some desktop environment available you can open it with your browser by clicking on the file "config_builder.html" or just copy/move this folder to your Web Server root directory - it's quite useful and well documented.

You need to edit the configuration files according to your needs. We'll use basic sample configuration files for server and transcoder that works with minimal configuration.

**sc_serv_basic.conf:**

```
logfile=logs/sc_serv.log
w3clog=logs/sc_w3c.log
banfile=control/sc_serv.ban
ripfile=control/sc_serv.rip
password=testing
adminpassword=changeme
streamid=1
streampath=/test.aac
```

**sc_trans_basic.conf:**

```
logfile=logs/sc_trans.log
calendarrewrite=0
encoder_1=aacp
bitrate_1=56000
outprotocol_1=3
serverip_1=127.0.0.1
serverport_1=8000
password_1=testing
streamid_1=1
streamtitle=Test Server
streamurl=http://localhost
genre=Misc
playlistfile=playlists/main.lst
adminport=7999
adminuser=admin
adminpassword=goaway 
```

A couple of important settings from both files

- **"maxuser=32"** - "The maximum number of simultaneous listeners allowed (The default is 32 so you don't have to specify this option). Compute a reasonable value for your available upstream bandwidth."

- **"password=testing"** - "Password used by sc_trans or the Winamp dsp plug-in"

- **"adminpassword=changeme"** - "**Admin** password required to perform administration via the web interface to the server."

- **"portbase=8000"** - "This is the IP port number your server will run on. The value, and the value + 1 must be available."

**Note:** If you've purchased MP3 encoding license you have to specify it in the transcoding config file

**"unlockkeyname=Transcoder MP3 unlock name"**

**"unlockkeycode=Transcoder MP3 unlock code"**

For the testing purpose we used the sample MP3 file located in "music" directory and the default play-list "main.lst"

At this point you can go through the settings and change them to what you want or you can save and start SHOUTcast and it will work perfectly.

So it's time to fire up your radio station

First we need to start SHOUTcast server and then transcoder. You can use separate terminal windows/tabs or software like "screen", the syntax is similar (./sc_serv <config_file>)

    ./sc_serv  sc_serv_basic.conf

and

    ./sc_trans  sc_trans_basic.conf

**Note:** You can also run both scripts as a daemon **"./sc_serv daemon <conf_file>"** or **"./sc_trans daemon <conf_file>"** (It maybe necessary to use absolute paths).

Visit http://localhost:8000 to see the song title and information - which is live information with a total number of listeners. If you login, you will be able to see users connected and IPs with the ability to kick them. To login, the username is **admin**, and the password is your admin password, or if not set, your broadcast password.

On the website, you will find a link to the listen.pls file that clients can download and play in Foobar, Winamp or other Shoutcast enabled players.

**Note:** The SHOUTcast streams are normally delayed 30 seconds to listeners, so people listening will hear what you hear live about 30 seconds after the fact.

To kill the shoutcast server or transcoder, type in **ps aux** and find the **PID** of **./sc_serv** or **./sc_trans** and then in your next command type in **kill -9 8874**, where **8874** is the **PID** of the **sc_serv** process. This will kill your SHOUTcast. Keep in mind that SHOUTcast will not start at system boot, so you will manually need to restart it. Also, the SHOUTcast server uses approximately 32MB of memory while running.

#### Installing Winamp on Windows and streaming to a SHOUTcast server

Now lets get Winamp running on Windows so we can stream!

Winamp is a media player for Windows-based PCs, written also by Nullsoft, for more information and to download this software please visit http://www.winamp.com/

**Install Winamp**

The installation is pretty straight forward and it's just a few simple clicks

**Note:** Configure available options to meet your needs.

- Download and install the SHOUTcast DSP Plug-In for Winamp. By default it will be installed in the same directory as Winamp

- Start Winamp > go to **Options** > **Preferences**, then in that window scroll down to **Plugins** > **Dsp/effect**. Select Nullsoft SHOUTcast Source DSP 2.2.3 then Configure active plug-in

- Find the new window that poped up, in it, go to the output tab and fill in the address: this is your server URL or IP address, port, and password. You can click on yellow pages and enter in personal info as well. If you make your server public, you will be listed in the SHOUTcast directory where anyone can find your server.

- Go to the encoder tab, and the encoder type should be MP3 encoder, and the settings should be whatever you want. In this example we're using 128Kbps, 44.1khz stereo.

- Go back to output and click on connect and it should be streaming to your server. Use Winamp to play a song of your choice. You can simply go to your now playing page in the main Winamp player window and open a song to play that's on your computer.

To broadcast your voice on mic, go to the input tab of the SHOUTcast source window and select sound card as input and push to talk (or lock if you want to click once to talk then click again to turn off your mic). Remember after done talking, switch back to Winamp as the input device to stream music playing in the player.
