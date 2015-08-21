---
layout: post
title: Snowflake Developer Roundup - January 2015
---
I tried to save the first devlog of the new year to be when the API is actually finished, but seems like I couldn't wait until then. Still, we're thiiiiissss close to having everything complete, or at least booting a ROM, so I'm going to blog about it.

Documentation
-------------

It's done! Pretty much all the interfaces are documented! Since you should be using the interfaces anyways and not the implementation, I've chosen to refrain from documenting nearly everything in `Snowflake` and instead put the documentation in the `Snowflake.API` assembly. IntelliSense still picks it up properly, as long as you keep your variables to the interface and not the implementation.

Plugins
---------

The DAT Identifier plugin and the TheGamesDB Scraper plugin have been properly refactored into their own projects without polluting the main solution file. They also build against the base implementation now, and don't require calling into CoreService.LoadedCore, instead a reference is injected into the constructor when MEF loads it. I might split off the `Service` namespace into it's own assembly later as well, plugins should never have to call into that.

RetroArch Bridge
----------------
It works, kind of, and only with bSNES. It can run a ROM file and compile configuration options just fine. Its still missing things like a fullscreen toggle but I guess we'll get to that later.

Configuration Flags
-------------------
This is kinda messy. SQLite is a poor format for the type of unstructured non-relational, configuration data that flags will have to be, so this will be changed into a simple file store that will load up the flags on request.

Controller API
--------------

While I had a controller API that worked alright, it was really messy and ambiguous. It assumed support for only XInput controllers and a keyboard and the way it coupled the controller, platform, and input device together in one entry was not ideal. I'm in the process of rewriting this API to be more flexible, with the caveat that it now requires a platform-dependent library that has to be called, thankfully this is easy enough to implement across platforms.

How this new API will work is that the controller ports, and what controllers are available will be defined as part of the platform definition. However, the actual controller definition itself is separate. This decouples the controller layout from the input device. Mappings are now independent-per-input device, and maps to Snowflake's `KeyboardMapping`  and `GamepadMapping` abstractions. The device name and ID is stored in the mapping and are no longer stored in a database, instead it's now a simple json configuration file per controller. The `ControllerPortsDatabase` instead of storing which controller definition is "plugged in" to which controller, simply stores which physical input device is plugged in. The emulator plugin can now be certain whether an input device supports which input APIs, and so forth and can generate controller configs accordingly.

..or so it will work. This is still WIP, with many parts yet to be implemented. So far I've split apart the platform definitions and controller definitions, and implemented an interface to get device information across platforms (only Windows at the moment due to my lack of Linux machines)

HTML Bootstrap
--------------

I've decided to move away from node-webkit (now nw.js), to atom-shell, which GitHub uses for their atom-editor. node-webkit didn't play along nicely with Polymer and window-context exceptions, which would crash the node context. Atom enforces further separation between the browser javascript and the node javascript. An error that happens in the webpage that normally would be ignored will continue to be ignored.

The Javascript API will also be rewritten, whatever was originally made of it.

Build Services, Linux and Unit Tests
------------------------------------

I've spent the past two days getting xunit and AppVeyor to play nicely together, which seems to be working now. Travis-CI builds are now working as well, they're failing because xunit 2 doesn't seem to support Mono yet, but the project builds, so there's that. I'd still like to wait for .NET Core and port it to that but it seems that's a long way off. At the very least, Snowflake has a test framework set up, so tests can be trivially added and run on CI services. I've set up code coverage information to be submitted and tracked on Coveralls.io every AppVeyor build to track Unit Test development progress. Also, something neat I've done is to get AppVeyor to generate and make available as an artifact doxygen generated documentation every build, so the corresponding documentation for each build is now available.

Stone
-----
I've drafted a standard called [Stone](https://github.com/SnowflakePowered/stone) that defines the database layout that Snowflake uses as well as consistent platform identifiers. It would be pretty cool if a whole bunch of emulator frontends used the same database schema, but I expect that part of the spec to go the way of [xkcd #927](http://xkcd.com/927/) just due to the way different frontends handle things though. Still, Snowflake will be sticking to this schema for the forseeable future.

However, I am really pushing for the [standardized, consistent platform identifiers](https://github.com/SnowflakePowered/stone/blob/master/Stone-platform.md). Across the whole emulation and retro gaming scene there is no single, agreed-upon way to refer to a certain platform. The NES could be the same as "Nintendo Entertainment System", or "GoodNES" for the GoodTools romset, or "Famicom" for the Japanese release, which is essentially the same chip. Oddly enough, we have region identifiers that are standardized. "U" means "United States", "E" means Europe, etc, this is consistent across Redump, No-Intro, GoodTools, etc. On the other hand, I've seen the Super Nintendo Entertainment System be referred to as "Super Nintendo (SNES)". This is fine for humans and all, but its not consistent at all. What I've attempted to do is assign consistent IDs for all but the most obscure consoles and this may come across as a little cocky but I really hope that the IDs are adopted, if not Stone-platforms then another set.
