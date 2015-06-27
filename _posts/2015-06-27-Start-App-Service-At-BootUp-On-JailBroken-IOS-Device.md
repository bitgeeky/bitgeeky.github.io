---
layout: post
title: Start App Service At BootUp On JailBroken IOS Device 
date: 2015-06-27 07:19:35 UTC
updated: 2015-06-27 07:19:35 UTC
comments: true
categories: ios jailbreak launchdaemon homepage
---

### This post is for Jail Broken devices and tested on IOS 8.1.2. ###


To start an application or service everytime on device boot up just follow these simple steps:

1 Install OpenSSH on your jailbroken device (through Cydia).

2 Write a boot script for your application or service.
This can be anything as simpler as a bash script or anything that launches your app.

3 Write a plist for application. An example plist for application is:

```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple Computer//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
        <key>Label</key>
        <string>com.example.BackgroundService</string>
        
        <key>ProgramArguments</key>
        <array>
        <string>/path/to/program/eg/bin/bash</string>
        <string>path/to/launchscript/eg/root/BackgroundService</string>
        </array>
        
        <key>RunAtLoad</key>
        <true/>

        <key>StandardErrorPath</key>
        <string>/var/logs/BackgroundService.log</string>
        
        <key>StandardOutPath</key>
        <string>/tmp/BackgroundService.log</string>
</dict>
</plist>
```
You can name it anything for example `com.example.BackgroundService.plist`

4 Copy the `plist` file to `/Library/LaunchDaemons/` directory.

5 Load the `plist` using:

```
$ launchctl load /Library/LaunchDaemons/com.example.BackgroundService.plist
```


That's it ! Now every time you boot up the device your service will start running automatically.
