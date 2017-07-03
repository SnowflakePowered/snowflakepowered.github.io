---
layout: post
title: Snowflake Progress Report - June 2017
---

This month's main attraction is the new Modules API that will serve as the basis for how Snowflake is developed and packaged in the future. There has been some UI progress as well; but if there is one thing I've learned this month, is that frontend developement is *very difficult*. Nevertheless, there has been some good progress in some of the core elements like the home view. Finally, there have been some improvements in filename parsing to make it more accurate than the previous regex-based method.


**If you know how to inject an overlay CEF on top of an arbitrary OpenGL/DirectX surface like the Steam Overlay, [please let me know!](https://github.com/SnowflakePowered/snowflake/issues/235)**

## **UI Progress** 

While the UI is not yet usable, I did get a lot of parts of the Material Design theme done in isolation these few weeks.

![main view](http://i.imgur.com/xybB3bS.png)

This will be the "Home", or the main view of the Material Design theme; the big red header section is meant to display some information about the currently selected platform. This view might have been one of the most difficult ones to write; a huge challenge was figuring out how to performantly display over a hundred games. Fortunately, [react-virtualized](https://github.com/bvaughn/react-virtualized) helped a bunch, but there is still some jankiness when there are over 300 games; I couldn't figure out how to optimize this yet so I'm hoping React Fiber will smooth this out once [it's ready](http://isfiberreadyyet.com/).

![platform view](http://i.imgur.com/hh63dzP.gif)

This is how one would select a platform to view games for; instead of having a sidebar like a lot of desktop-oriented frontends, I decided to go with a separate menu instead. Perhaps this might be too mobile-like but for now this is how it'll stay for this theme at least.

![game view](http://i.imgur.com/iaHwxte.png)

This is very much a work in progress; the game view is where users can set launch options for their emulator, view and edit details about the game, and change their controller configuration. The UI toolkit I'm using, [Material-UI v1](https://material-ui-1dab0.firebaseapp.com), doesn't support all the components I need to make this work properly, so I will need to work around that by using their older version. Please ignore the fact that there is a Super Mario Bros. image on top of a Super Mario World background..

Currently, React lets me develop each part in isolation which makes development a lot easier when I don't have to worry about side effects. Eventually I will have to hook everything up and decide how one view links to another, this is called *routing*. I'm still thinking very deeply on how to do this properly, and how to store the currently selected game and platform.

Note that most of the UI work is currently done in the [remoting](https://github.com/SnowflakePowered/snowflake/pull/246) branch, and has yet to be merged to master.

## **Module Loader and Plugin API Rewrite** *(PR [#249](https://github.com/SnowflakePowered/snowflake/pull/249))* &mdash; More reliable plugin system.

Since the port to .NET Core, plugin loading has been extremely fragile; it was basically held together with duct tape and gum to port it as soon as possible so that work could be resumed quickly under the restrictions of the new SDK. What used to work under the Windows-only .NET Framework no longer did reliably when cross-platform was taken into consideration, and while Snowflake was from the start built with cross-platform support as an eventuality, the plugin loading wasn't something I was especially concerned with.

Not only have I completely rewritten the plugin loading backend, there have been some changes made to the way plugins have to be packaged to make things easier to manage and install. Before, plugins and all their dependencies were simply thrown into the *plugins* folder. Proper load order was basically hoping for the best, which isn't quite reliable especially if you have pluginsn depending on plugins. Now, plugins are packaged as *modules*, which are loaded individually and have separate folders to put their dependencies in. Each plugin is loaded in isolation to other plugins, so dependency load order is always correct. 

![plugin format](http://i.imgur.com/n0ePzFC.gif)

Another change made to the plugin system are how they are initialized. Before, plugins were registered with the plugin manager through a container that had access to the entire instance of Snowflake. There was no way to tell what plugin had access to what services, and no way to tell if a plugin can even be loaded properly when a service doesn't exist. The new method requires all plugin containers to list which services they require, so that the plugin loader when to load a plugin, when all of its required services are available. If the services are never available, the plugin will simply never be loaded. 

With the new changes in plugin loading, a lot of service implementations have been moved outside of the core framework. For example, the plugin manager implementation is now implemented as its own plugin (technically a service module). When launching Snowflake, only a few essential APIs (eg. a service that gives the application data path, a service that allows more services to be registered, etc.) are provided initially, the rest are loaded externally as their own separate service modules. This is all towards the goal of making Snowflake as moddable as possible.

For most people looking to extend Snowflake, the so-called *Plugin API* should be sufficient, which relies on the plugin manager to provide it certain things like loading metadata from a resources folder, a configuration database to store options, etc. The lower level *Service API* allows *modules* to register services that other plugins can use and hook into; this is how a bunch of APIs that used to be explicitly instantiated are now implemented, like the plugin manager. By keeping implementation details like this out of the core framework API, this reduces the size and complexity of a plugin greatly. This is very similar to a programming concept called "dependency injection", which is used to reduce complexity by making what a certain object requires very explicit. 


## **Tokenizing Filename Parser** *(PR [#250](https://github.com/SnowflakePowered/snowflake/pull/250))* &mdash;  ROM filename parsing improvements

While Snowflake also uses advanced techniques like looking into the ROM header to get information to scrape from, oftentimes the ROM's filename also provides useful information. Snowflake used to glean the game's title, year, and country using regular expressions, but this wasn't very accurate. 

The new method splits up the filename into "tokens", or a word with a specific meaning. Since ROM filenames often have flags in brackets or parenthesis, these flags are extracted individually, and classified by meaning as if the ROM was simultaenously named with the GoodTools convention, the No-Intro convention, and the TOSEC convention. Snowflake then picks the convention with the best result and uses that to determine the title, the year, country, languages, etc. The old method would fail on a lot of examples, but this new method proves accurate for [quite a few test cases](https://github.com/SnowflakePowered/snowflake/blob/master/src/Snowflake.Framework.Tests/Romfile/StructuredFilenameTests.cs).

## **Looking forward to the next report...**

For the month of July, most of the work will be in the user interface, and hopefully getting it up to a usable state so that games could at least be added to the frontend, if not yet launchable. I also hope to work on the packaging side of things to make Snowflake easier to install and update. I'm also looking into launching a Patreon very soon, **please let me know what you think**, no matter if you plan to donate or not. Some income would help a lot with hosting fees and infrastructure; currently I'm using a lot of free services for nightly builds and such, but I have some big plans that I can't possibly implement without any money. 

I'm also thinking about a complete redesign of the website and logo; a lot of the visual identity stuff is getting dated seeing as how they were designed two years ago, and I like to think my design skills have improved since then. This will probably not be done in July though, as getting the UI functional takes immediate priority.
