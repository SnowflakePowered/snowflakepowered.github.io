---
layout: post
title: Snowflake Developer Roundup - April 2015
---

There hasn't been as much progress as I would have liked from February to April, but work is still continuing at a steady rate. There's been few, but very large improvements that I'd like to talk about. That, and I finally have a screenshot of a working incarnation of the final UI.

Material Design UI
------------------

Since my last devlog in February, I've gotten my UI up and running, and basically displaying information regarding a game. Currently it can performantly display a large amount of games, and scrape them from just the file. I'm still missing things like changing settings and controller configuration from within the frontend, and a file-browser to add games. (Right now, you have to manually type in the path for each ROM), but it's coming along fine. When `paper-elements` are ported to [Polymer 0.8](https://www.polymer-project.org/0.8/), work will begin on porting this theme to Polymer 0.8 for more performance increases.

![screenshot](https://camo.githubusercontent.com/2a41d2013108677782ae8e30007989c46ed93d67/687474703a2f2f692e696d6775722e636f6d2f6455625a536b312e706e67).

Identifier Improvements
-----------------------

Snowflake's scraping system works by first inferring available information from the ROM file itself, such as the filename, the CRC32 hash, or the game name by looking up a dat from No-Intro, before sending that and querying something like [TheGamesDB](http://thegamesdb.net/), and comparing the results to get the best match. This works very well, but there's always been a problem with scrapers only being able to use a single 'identifier' value. I tried to solve this before by passing all the values in a dictionary, keyed on the identifier plugin name. However, this means the scraper would have to know what identifier plugins are available beforehand, which isn't ideal.

What I ended up doing was passing additional metadata along with the raw information from the identifier. The scraper gets a list of the data, which includes the actual value, the name of the plugin, and the type of data it is. For example, if the identifier finds the title of the game, then the type of value is `GameTitle`. The scraper plugin can just look up all values that are `GameTitle`, and get the first one. Or if it needs a Disc ID such as for PSX or Wii games, they can look up all values that are `DiscId`, and compare the scraped results with the Disc ID to ensure it's the correct result. Snowflake will also support loading local data from JSON files for those that don't like re-scraping their library.

Goodbye MediaStore
------------------

MediaStore was a way for Snowflake to store images, videos and audio about Games and Platforms. &nbsp;It was a really complex system designed to be as generic as possible to incorporate the needs of storing media for both Games, Platforms and everything in between. As I was building the theme you saw above, I came to realize that the MediaStore system was too bloated and buggy a system. Platforms didn't need media data stored in the core, images of those should be handled by the theme. I didn't have any other pieces of information using it other than games, and if I'm only using it for Games, why not redesign it. MediaStore was also a piece of cruft in the [Stone](https://github.com/SnowflakePowered/stone) standard, a loose set of requirements meant to provide some standard ways of handling information throughout all emulator frontends. Although it's unlikely it'll catch on, the idea of a Snowflake-exclusive concept in the games database ran in opposition to Stone's ideals. And so, MediaStore was phased out of Snowflake and made obsolete.

GameMediaCache
--------------

With MediaStore gone, Snowflake still needed a way to store data like boxarts and game-clips. It should be simple to use, be loosely-coupled, that is, exist entirely outside the game database, and be designed for the needs of games. The GameMediaCache is a simple way to store certain types of media related to games. It can store

 * BoxartFront
 * BoxartBack
 * BoxartFull
 * GameFanart
 * GameMusic
 * GameVideo

and only these 6 items. Every game automatically has cache created for it on scrape-time, and its keyed on the game uuid, so all the theme needs to know is the id of the game, no special, extra key nescessary. To further ensure ease-of-access, GameMusic will only accept mp3, wav and ogg, and GameVideo will only accept h.264 mp4 and webm files.

GameScreeenshotCache
--------------------

MediaStore was also a way for games to store screenshots, but it was never really built for that purpose. GameScreenshotCache was built so that emulator-bridge plugins can easily save screenshots to a central directory for Snowflake to access. Screenshots are uniquely named with the time the screenshot was added, and are saved as PNG regardless of the original format of the screenshot. I hope to have something like a `FileSystemWatcher` watching a directory that emulators save screenshots in, and copying it into the GameScreenshotCache

Accessing the new Caches: GameCacheServer
-----------------------------------------

Snowflake was intended to have the UI frontend be done in the browser. Certainly it doesn't restrict other ways of interfacing with the core backend (the part that handles scraping,.game-database and emulators), it does give provide some convenience features for access across HTTP.

The GameCacheServer replaces the old FileMediaStoreServer and is an easy way to access the GameScreenshotCache and GameMediaCache. Image files are always served as PNG files with the mimetype `image/png`, and since music and video are restricted to browser-friendly formats, there won't be any trouble utilizing such resources. GameCacheServer also supports on-the-fly resizing of images to reduce any client-side performance issues. For more information about the GameCacheServer, refer to the [Pull Request notes for PR #110](https://github.com/SnowflakePowered/snowflake/pull/110)

Onwards
-------

I haven't planned on making such drastic changes to the API, but it came out of necessity. MediaStore was full of cruft and bloat and was in hindsight a bad idea. I much prefer a more structured way of saving this data in GameMediaCache, and GameMediaCache is very loosely-coupled, I could remove it without breaking API as I had to when deprecating MediaStore.

Onwards, we're looking at finalizing the features of the UI and finally putting all this stuff into atom-shell rather than debugging through Chrome. Everything is on track for a July or August developer preview release, and I'm very excited. If there's time, I hope to get more RetroArch cores wrapped, and a Dolphin wrapper as well, as one of my original goals in Snowflake was to have a nice-looking frontend to Dolphin.
