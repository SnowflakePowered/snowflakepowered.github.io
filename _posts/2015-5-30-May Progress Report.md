---
layout: post
title: Snowflake Progress Report - May 2015
---
Snowflake has come a long way from a bunch of hacky Python scripts. 2 years since I started working on a frontend, I’ve finally managed to build something that actually resembles one with C#, HTML5 and Javascript. Snowflake finally comes to a semblance of reality, an app that a person can actually use, and I’m extremely excited to show it off. Everything you see, and more will be available by **July’s alpha release.**

![screenshot](http://i.imgur.com/66VDHTW.png)If you’ve been following Snowflake’s development this screenshot should feel familiar. This Material Design theme will be Snowflake’s very first theme, built to feel familiar and intuitive. Once I’ve established how everything fits together, **I’ll reuse parts of the code to build an XMB-like 10-foot interface.** This screenshot just shows off the dynamic colour-schemes I have going on that determine the colours by the cover art.

I’ve made some optimizations to this feature by caching the colours, so it only has to calculate the colours once; this resulted in a speedup of at least 10FPS during transitions going by Chrome’s dev tools.

![flags](http://fat.gfycat.com/FabulousEmptyDragon.gif)

One of Snowflake’s most important features is setting emulator options inside the interface. When you make a change in setting to a certain game, it’s marked _customized_ for that game, unless you reset it to default. Every time you access the settings, any option not marked _customized_ for a game will re-sync with the defaults. If you make a change to a default, those changes will propagate to every single game which doesn’t have the option marked customized.

The system is remarkably simple. You can set an option, and the emulator plugin just flips a few switches, generates a configuration file and runs the emulator with the selected ROM. This will work with _any_ emulator that uses configuration files.

![controller-select](http://fat.gfycat.com/KeyTanFieldspaniel.gif)

You can also change controller settings inside Snowflake. How it works is, Snowflake stores a “definition” for a certain controller, that’s linked to a certain platform — the _NES_CONTROLLER_ is linked to the NES platform, etc. Then, for each controller, there’s a profile that’s linked with a certain input device. Usually the profile name is the same as the full device name, but for convinience, Snowflake gives the keyboard and 4 XInput controllers names that are easier remember than `Microsoft Xbox 360 Controller (for Windows)`.

![set-input](http://giant.gfycat.com/QuestionableWelltodoGrassspider.gif)

Click the input you want to change, and a dialog will pop up. Press the new key you want, save your changes, and your controller setting for **every emulator for that platform** has been changed, just as if you were using a core in RetroArch. Except instead, we’re just wrapping an EXE. And it’s easier to use.

**What’s left?** The cruicial parts I’m missing right now are

*   A way to set player 1, 2, 3, 4.
*   A way to change the default options for an emulator
*   Putting everything inside electron-shell instead of running it in Chrome.

The first two will be easy to implement. All I have to do is write the UI for that and hook it up to existing Javascript callbacks supplied by the C# core. The last part is the most important, but will be the most time-consuming to do. I need to be able to wrap everything up into one package, especially since having access to the filesystem is extremely important, which Chrome can’t give me. I’m not sure I’ll be able to make #3 by July, so the alpha release won’t be as polished as I’d hoped. However, I’m going to work on putting in all the features first before working on that.

There’s also the issue of Polymer 1.0\. I’m currently using an old set of Polymer components that aren’t foward compatible. I’ll have to port everything over to the new version of Polymer once they’ve ported all the elements in. I’m expecting to do that after the alpha release in July.
