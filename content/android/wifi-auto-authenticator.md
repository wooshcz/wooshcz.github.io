---
title: "Wifi Auto Authenticator"
date: 2022-06-20T22:07:17+02:00
draft: false
---

## Features
This app simplifies and automates the login process on the Cisco Web Auth captive portal that is used on some WiFi networks. The app detects changes of the network connectivity and resolves the captive portal redirection on a selected WiFi network with an automatic login function.


## Rooted phones
The app can automatically switch on/off mobile data on your phone during automatic login to bypass the built-in captive portal detection inside some versions of the Android OS. There is also a manual toggle to turn the mobile data on and off.


## Permissions
- Coarse Location is necessary on Android Marshmallow to perform WiFi scanning.
- It is necessary to access and change Wireless State to perform the scanning and turn on the WiFi.


This app stores the Web Auth portal password in plain text because it is then sent in the HTTP request during automatic login procedure. Be aware that the HTTP requests do not have to be secure if the Web Auth is running on un-encrypted HTTP. This app does not validate the server SSL certificates.

[Link to Google Play Store](https://play.google.com/store/apps/details?id=com.woosh.wifiautoaut)