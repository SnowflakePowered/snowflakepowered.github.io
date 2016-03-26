---
layout: post
title: Snowflake Developer Roundup - February 2015
---
Since my previous devlog ['The Home Stretch'](http://blog.ronnchyran.com/post/109230804563/snowflake-devlog-2015-1-26-the-home-stretch), I've basically implemented most if not all of the base C# API. At this point, most of the behaviour of the backend is finalized, so there's only a <del>few things to talk about.</del>

### New Controller API

The new controller API is complete and is a large improvement from what I had before. As described previously, not much has changed except for the fact that Snowflake now stores the input device in it's port database. I have some simple shims such as `KeyboardDevice` and `XInputGamepadDevice` for keyboards and XInput gamepads on Windows, but otherwise it'll store the device name of the input device. I've come up with a rather nice solution that abstracts data from udev or DirectInput into the [`IInputDevice`](https://github.com/SnowflakePowered/snowflake/blob/master/Snowflake.API/Emulator/Input/InputManager/IInputDevice.cs) class, and a library on each respective platform is called to handle this. Since EmulatorBridge plugins are not expected to be platform-agnostic, they should be able to tell whether they're running on Linux or Windows and use the information accordingly. I'm actually really excited about this part of Snowflake, and I'll be doing a type of technical demonstration video on this later.

### Events

I've defined and implemented most of the C#-side events in the API. However they've yet to be hooked up besides one or two, so while they can be referenced, nothing will actually be called yet. Hooking up the events is pretty trivial and there are other things I want to focus on before I do that however.

### New IScrapeService

Snowflake uses an interface called the `IScrapeService` to scrape information about a ROM. How scraping in Snowflake works is pretty simple: a plugin called an *identifier* tries to extract information from the ROM file itself, such as by matching the CRC32 to the No-Intro database, or through the ROM headers. Then, it would pass that single piece of information to whatever scraper plugin is marked as 'preferred' for that platform. Previously, you would give it a filename, it would run the preferred identifier on it and give that to the preferred scraper. There are a few issues with that; it assumes that the identified information is the game title, and it automatically picks the scraped information given the Levenstein distance from the scraped result title and the identified information. So, if you have a disk ID identifier give something like `RSBE01`, that sure as hell doesn't look like `Super Smash Brothers Brawl`. I've made some improvements to the IScrapeService to fix these issues.

Firstly, *all* the identifiers that support the ROM's platform are now called instead of just the one. The information is passed to the scraper in a dictionary with the identifier plugin name as the key. This gives the scraper as much information as it needs, as well as telling the scraper what identifier provided whatever information about a ROM file. The scraper now has access to more information than just one string.

Secondly, the scraper plugin is now responsible for sorting their results for relevancy. This is pretty simple, just a call to a Levenstein distance algorithm and an `OrderBy` is sufficient, but I've decided to move this to the plugin-side from the scrape service for reasons that will be apparent below.

Thirdly, scraping is now a 3 step process that involves the client-side calling the backend C# side. [`GetGameResults`](https://github.com/SnowflakePowered/snowflake/blob/master/Snowflake.API/Service/IScrapeService.cs) will be called given the filename, and it will run and pass on identified information to the scraper. This method will return the `IGameScrapeResult` objects to the client-side. Then, the client-side will call `GetGameInfo` with the filename and the result id. This will return the `GameInfo` object for the game complete with scraped information. At last, the client-side will call `AddGame` and add the game to the database. This provides the oppurtunity for the HTML client-side UI to choose which result is the most accurate, or simply pick the first, most relevant, as it has been sorted previously by the plugin.

### Ajax API

There's been some slight changes to the Ajax API to make it a bit easier to consume. The first is the addition of the `Success` parameter, indicating whether a request has succeeded or failed/errored. Instead of returning undefined, a failed request will now return the full exception stack trace as a JSON object, so as to give the client-side some more idea of what went wrong. Lastly, in regards to the API, the ['AjaxMethodParameterAttribute`](https://github.com/SnowflakePowered/snowflake/blob/master/Snowflake.API/Ajax/AjaxMethodParameterAttribute.cs) has been implemented just to provide some metadata to the programmer as to what parameters the ajax method uses. It's not mandatory, but it's encouraged. Again, all parameters are consumed with GET; Snowflake does not support POSTing data.

### StandardAjax

On top of the Ajax API, the Snowflake ['StandardAjax'](https://github.com/SnowflakePowered/snowflake/tree/master/Snowflake.StandardAjax) library has been mostly implemented with a few exceptions here and there. This replaces the old Ajax.SnowflakeCore plugin, and is used the same way: it exposes GET endpoints for the Ajax HttpClient to control and get information from the C# backend.

### snowflake.js

snowflake.js wraps the StandardAjax API in a bunch of XMLHttpRequest calls and promises. It's still in it's early stages, but development will be happening in CoffeeScript (because CoffeeScript keeps me from accidentally setting off a few nukes when trying to get the list of platforms) in the ['snowflake.js repo'](https://github.com/SnowflakePowered/snowflake.js). It uses CommonJS `require` internally, which is fine for atom-shell based apps which I will be using, but in a plain browser, I'm using `mr` (montage-require) to shim the `require` function for testing. You'll need to run an http server from the root directory for it to work, I suggest node's http-server module.

### UI

I'm trying to get a basic, unstyled HTML testing UI up as soon as possible so I can show some demonstration videos of what's possible with snowflake. Otherwise, I'll pretty much be using Material Design in the form of Polymer's `paper-` elements, and cloning the lakka.tv interface for my big-picture theme. Any designers are welcome to help :)

### Fure-kun and IRC

[Fure-kun](https://github.com/SnowflakePowered/fure-kun) is a side-project-thingy Hubot that gets Travis and Appveyor statuses for Snowflake and can trigger rebuilds. If you're wondering about the name, the katakana under the Snowflake logo (&#12473;&#12494;&#12540;&#12501;&#12524;&#12540;&#12463;) reads "sun&#333;fur&#275;ku", ergo, "Fure-kun". Theres no license up on the repo because I'm too lazy, but if you want the source, treat it as MIT.

**Join me on IRC at `irc.stormbit.net#snowflake` or [webchat](http://iris.stormbit.net/?channels=#snowflake)**
