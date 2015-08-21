---
layout: post
title: Snowflake Progress Report - August 2015
---

First off, apologies for the delay in releasing the alpha preview. Originally, I planned to release earlier, but as I went on, I felt Snowflake is still not ready for use. I want to deliver you the best frontend ever, even in alpha. Only recently were some features added, and there still aren't enough emulators that have had plugins written, which is taking more time than anticipated. The main theme still has a few bugs and unimplemented features, and overall it's lacking in polish that would've been great for a preview. That doesn't mean you can't see what I've been up to these past months though!


You can contact through [reddit](http://reddit.com/u/ron975) or by email at [ronny@ronnchyran.com](mailto:ronny@ronnchyran.com). Without further ado, let's take a look at some new features since the last showcase.

Better settings
----------------


![Better Settings](http://i.imgur.com/qgVGi2U.gif)

Changing settings has been improved greatly. The previous "Customized" system has been completely overhauled to be more explicit with a locked (set to default settings) and unlocked (customizable) system.  Loading and syncing settings has also been optimized, it now loads and refreshes twice as fast, and doesn't randomly lose customized settings anymore as it did before.


One device, one gamepad
-----------------------


Snowflake used to require every single device be separate in terms of mapping controls. This would've meant having to set different button mappings for every controller, for every device. You would be setting the 'A' button fifteen times just so you could play _Super Mario Bros. 3_ and _Sonic &amp; Knuckles_ the same way.

![Simple Control](http://i.imgur.com/MIEOg9e.png)


This feature has been reworked to use a system more similar to RetroArch's. For every input device there is only one set of mappings to a "virtual" gamepad that is worked down the each emulator's controls. The A button is now the same button across the Genesis, the PlayStation (err.. ![Cross](https://upload.wikimedia.org/wikipedia/commons/c/c2/PlayStationX.svg) button), and the Gamecube.

PLAYER ONE &mdash; Insert Controller
------------------


![Port Settings](http://i.imgur.com/jOi3bVa.png)

You can choose which device you want to use for which player, just like an actual console. <small><del>Now you can beat Psycho Mantis</del></small> For now, Snowflake works only with the keyboard/mouse and XInput controllers. Support for DirectInput and udev (Linux) will come soon after the alpha.

More and more emulators
-----------------


The most important part of an emulator frontend is the emulators. Many more emulators have already been adapted to Snowflake's powerful plugin system, like [Mednafen PSX and Genesis Plus GX](https://github.com/SnowflakePowered/emulator-RetroArchBridge). A Dolphin plugin is [already coming together](https://github.com/SnowflakePowered/emulator-DolphinBridge). Snowflake's plugin system is extremely flexible; the Dolphin plugin will detect whether or not you have Wii remotes plugged in and automatically override any emulated Wiimotes.

Spiffy installer
------------------


![installer](http://i.imgur.com/IMsdNxZ.gif)
Snowflake is going to be easy to install. When you first start it up, Snowflake will grab a list of plugins from a plugin repository and automatically download and install them. We're still working on the downloading part (the alpha will have to come with all plugins pre-installed), but Snowflake makes both the first setup, and keeping up to date, extremely easy.

More to come
---------------


We're quickly approaching Snowflake 1.0 Alpha. The dream I've had for [2+ years](https://github.com/RonnChyran/Snowflake-old) and [3 rewrites](https://github.com/SnowflakePowered/snowflake-py) is finally coming to fruition, but it's not done yet. I have big plans in mind, including cloud saves, automatic CUE generation for PSX games and a 10-foot theme based on [Lakka.tv](http://www.lakka.tv/). If you have any more ideas or features you want to see in Snowflake, feel free to contact me through [reddit](http://reddit.com/u/ron975) or by email at [ronny@ronnchyran.com](mailto:ronny@ronnchyran.com). I can't wait to see what I can show off by the Alpha.
