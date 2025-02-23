---
layout: post
title: Streaming PC games to your TV
date: 2025-02-19 16:25 -0500
categories:
- Projects
tags:
- tech
- gaming
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
- [Sunshine](https://app.lizardbyte.dev/Sunshine/?lng=en-US), an open-source host (server) for game streaming

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

# Optional Stuff

## Running Moonlight at startup

The 'most correct' way to run something in the background and keep it running is by making a daemon/systemd service for it.  We'll create a the following file and place it in `~/.config/systemd/user`

```
[Unit]
Description=Moonlight Client

[Service]
ExecStart=/usr/bin/moonlight-qt
Type=exec
Restart=no

[Install]
WantedBy=default.target
```
{: file="moonlight.service"}

We then need to run a few commands to enable the service:
```shell
systemctl --user daemon-reload
systemctl --user enable moonlight.service
```

You can confirm it's running with
```shell
systemctl --user status moonlight
```

Now, whenever you have a session open for the user, moonlight will be running.  If you want to have it start moonlight regardless of a user session running (ie, you don't want to have to SSH in) you can do that with `sudo loginctl enable-linger $USER`. This command makes the user's systemd instance independant from their sessions.


## Pairing a Bluetooth controller to your Pi

1. Enter `bluetoothctl`
2. Turn the agent on with `agent on`
3. Scan for nearby devices with `scan on`. This will show nearby devices' names and mac addresses
4. Pair the device you want with `pair <mac address>`
5. To connect to a previously paired device, use `connect <mac address>`.  Additionally, use `trust <mac address>` to trust the device. If you'd like the controller to connect automatically (without using the commandline each time) we'll need to set up another service.

### Automatically reconnecting Bluetooth devices

Unfortunately this way is a little hacky and I haven't been able to find a 'real' solution.  First, we'll make a script that continuously tries to connect to our trusted devices.  Then we'll add a cronjob that starts the script on boot.  I initially tried making another service instead of a cronjob, but was unable to get it working.  This way when we turn the controllers on, the pi will already be attempting to connect them.  My bash script looks like this:

```shell
#!/bin/bash

devices=$(bluetoothctl devices | awk '{print $2}')

while true; do
for device in $devices; do
	bluetoothctl connect $device
done
done
```
{: file="connectTrustedBLEDevices.sh"}

Here's what the script is doing.  First, we get all of our trusted devices and use awk to pull their mac address.  Then we start and infinate loop and go through each device and attempt a connection.  If the device is already connected it still runs the connect command, but I haven't experienced any performance issues due to that.  

I have this script saved in my home directory under the scripts folder. Also be sure it's executable, which can be achieved by running `chmod +x connectTrustedBLEDevices.sh`

Now we need to set up our cronjob. We'll run `crontab -e` to edit our user's cron file.  Then, place the following line at the bottom

```
@reboot sleep 300 && /home/<user>/scripts/connectTrustedBLEDevices.sh
```
{: file="crontab"}

This job first sleeps for 5 minutes upon a reboot then runs our infinate script to attempt to connect to the trusted bluetooth devices.

### Future Improvement
I'd like to figure out a better way to do this than running this script all the time. Ideally, I think using GPIO and hooking up a button to the pi could be a good solution.  Pushing the button could run the script for 1 minute to allow trusted devices to connect, then stop.

## Other Notes
- Sunshine comes with full desktop and steam pre-add, but you can [add other apps as well](https://docs.lizardbyte.dev/projects/sunshine/latest/md_docs_2app__examples.html)
