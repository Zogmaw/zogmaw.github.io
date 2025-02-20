---
layout: post
title: Streaming PC games to your TV
date: 2025-02-19 16:25 -0500
categories: [Projects]
tags: [tech, gaming]
---
# Problem

There are a lot of great couch co-op games that have come out recently, however there is a major issue with many of these games - how can you actually play them on your couch?  If you primarally play games on your PC you're probably in the same boat.  This is how I managed to stream games from my PC to my TV to get the full couch co-op experience

# Materials

This is what you'll need to replicate my setup.

Physical:
- Raspberry Pi 4 Model B (Model 4 is recommended)
- MicroSD card for the Pi
- Ethernet Cable (WiFi may work, but ethernet likely results in less lag)
- Micro HDMI to HDMI (Or whatever input your TV uses)

Software:
- [Moonlight](https://moonlight-stream.org/), an open-source client for game streaming
- Server

# Guide

## Setting up the Pi

1. First you'll need to install RaspberryPi OS Lite on your MicroSD card.  This is easily done using the [RaspberryPi Imager](https://www.raspberrypi.com/software/).  Within the settings you can also configure WiFi and SSH setup.
2. Plug the Pi into your TV and Ethernet, then SSH into it.
3. Install moonlight.  This can be done with `curl -1sLf 'https://dl.cloudsmith.io/public/moonlight-game-streaming/moonlight-qt/setup.deb.sh' | distro=raspbian codename=$(lsb_release -cs) sudo -E bash` then `sudo apt install moonlight-qt`.  You can then run moonlight using `moonlight-qt`.
4. Install an audio driver on the Pi with `sudo apt install pulseaudio`. Then use `sudo raspi-config` and navigate to 'Advanced Settings > Audio Config' and choose PulseAudio.  Now reboot with `sudo reboot`.  Finally, select the audio device with `sudo raspi-config` under "System Options > Audio"
5. To enable DS4/DS5 controller support, run `sudo usermod -a -G input $USER`

## Setting up your PC

1. First install [sunshine](https://app.lizardbyte.dev/Sunshine/?lng=en-US)
2. After installing and running, it should open up a web GUI on https://localhost:47990 and prompt you to create a username and password.
3. After authenticating, you're able to edit sunshine's settings.  The defaults are largely fine, so we'll go right to the Pin tab to pair with the moonlight client.
4. From the moonlight client (not the ssh terminal, your actual TV) you should see your PC in a list of servers.  Attempt to connect to it and it should give you a PIN.  Put that pin in the sunshine web GUI and give the moonlight client a name.
5. That's it! You should now be able to start a connection from the moonlight client to your PC to play games.