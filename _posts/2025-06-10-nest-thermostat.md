---
layout: post
title:  "Nest Thermostat E fails to connect to the Nest app (TD008 4060)"
date:   2025-06-10 12:00:00 -0700
categories: Nest iOS Google IoT
---

## Error codes

The Nest app on the mobile device might show one of the following error codes:

- TD006 (4003)
- TD007 (4072)
- TD008 (4060)

You could get one of these error messages: 

- "Your thermostat couldn't connect to Wi-Fi. Make sure that your Wi-Fi network is connected to the internet and that you entered the correct password."
- "There was a problem adding the thermostat to your Nest Account."

## tl.dr. Solution

**Use a different mobile device to connect to the Nest thermostat. If you've been using an iPhone, try downloading and using the Nest app on an iPad / an older iPhone / an Android phone.**

*The thermostat can be controlled by any device after successfully paired on the Nest app into a Google account.*

*You probably don't need to change any setting on your Wi-Fi router.*

## Findings and notes

* The Nest Thermostat E does support 5 GHz Wi-Fi networks. 
* It can also connect to Wi-Fi networks that have both the 2 GHz and the 5 GHz signals on the same SSID (i.e. the "hybrid" Wi-Fi networks).
* Go to "Settings" - "Network" on the Nest thermostat and try connecting to your Wi-Fi with the SSID and the password. \
  If this succeeds and the thermostat still fails to connect to the Nest app, this indicates the issue is around the connection between the thermostat and the phone (likely via Bluetooth LE), instead of the connection to the Wi-Fi router / Internet.
* *None* of the following resolves the issue for me, but it may be worth a try.

| Try this | Note |
| -- | -- |
| Reset *all* settings on the Nest thermostat. | This may be helpful to clear Schedule settings from the previous homeowner / apartment renter. <br>You need to select the hardware wiring again afterward. |
| Switch the Wi-Fi router to 2.4 GHz mode. <br>Set up a dedicated 2.4 GHz Wi-Fi network on the router and use it on the Nest thermostat and the phone. | |
| Remove the password on the W-Fi network. <br>In your router settings, use "No Security" instead of "WPA/WPA2-Personal" & "AES". | The Nest thermostat E does support WPA2 encryption on Wi-Fi networks. |
| Use the "802.11 b/g/n mixed" mode instead of the "802.11a/n/ac/ax mixed" mode on the Wi-Fi network. | Older Wi-Fi standards typically have better compatibility. |
| Specify 20 MHz as the Channel Width of the Wi-Fi network instead of "Auto". | 20 MHz is the standard channel width with potentially worst data transfer rates but better compatibility. |
