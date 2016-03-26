---
layout: post
title: Feature Preview - Heuristic Scraping
---

_This is part 1 of a new set of articles going into depth of new features in Snowflake running up to Snowflake API 1.0_

**The Pull Request featured in this article is [#170](https://github.com/SnowflakePowered/snowflake/pull/170)**

Scraping has always been a fuzzy and controversial field. Unlike other forms of media, games; ROMs in particular, are unique across platforms, have multiple naming schemes (if the ROM is properly named at all), and some are relatively large files, especially ISOs for more recent systems like the PS2 and the Wii. Unlike TV Shows or Movies which have a single centralized, community-maintained database, games can have multiple sources of Data with varying degrees of accuracy. The localization factor, coming from games brought over from Japan or other countries, and games sharing data between regions makes scraping games given only the data within the file, and the filename, a difficult process.

The Old Ways &mdash; Identifiers
--------------------------------
Once we have a ROM, how do we find out what game it is? Different games can have the same name, especially multiplatforms, and have different revisions. The obvious way is to simply look it up in a database. This was the thought process behind 'Identifiers', which ended up being almost mini-scrapers. Before data was sent to a scraper to get more information with, an identifier plugin would look up data from it's own little database, such as datfiles. Each identifier would put a single piece of information inside a list, and a scraper would have to know which identifiers were installed, and which pieces of data were available. For example:

*Super Mario Bros 3.nes* could have something like **`rom_crc32 = 0B742B33`**

And the scraper would have to know that it had access to `rom_crc32`. Of course, calculating the CRC is also very expensive for large files and would often take long. Another identifier would use pattern-matching to get the name "`Super Mario Bros 3`", but what if the file was named `smb3.nes`? The scraper would have no idea what to use. 

The problem with identifiers was that they were too multi-purpose and the way they worked limited them to only one string of output, that each scraper plugin would have to be aware of. This would have been super fragile: if a scraper plugin relied on a certain identifier plugin, and that identifier plugin wasn't installed, the scraper wouldn't work. If an identifier only worked on a single type of ROM, for example WBFS, then the scraper would have to be aware of that too, and customize their scraping routine. 

The biggest problem with identifiers was that the platform of the ROM had to be known beforehand. Of course, this is no problem with filenames like `.nes` and `.sfc`, but what about `.iso`, and `.bin`? There's no way to know the platform of game for these types of files. There were a few ways of dealing with this: There would have to be an identifier solely dedicated to finding out the platform of a certain file; or the user had to know which platform a file belonged to beforehand, and tell Snowflake that this ROM is an PSX ISO, or a Wii ISO, or a Genesis BIN and not a PSX BIN, for example.

This failed spectacularily if the given data was wrong.

Solving the name parsing issue &mdash; StructuredFilename
--------------------------------------------------------
ROMs nowadays commonly use 3 naming formats: GoodTools format, No-Intro format, and The Old School Computing Centre (TOSEC) format. If your ROMs are healthy, chances are your file is using one of those 3 formats. If we were using identifiers, there would have to be an identifier for each format, and the scraper would have to check all 3 identifiers, and since all 3 overlap slightly, scraper plugins would have no idea which one was correct.

StructuredFilename was a result of studying the differences in the naming formats, and normalizing them to give a proper game title that the scraper can look up. GoodTools, No-Intro, and TOSEC all use different ways to represent a region in their filename, so that was used as a discriminant. GoodTools uses their own abbreviations, and No-Intro uses full country names while TOSEC uses ISO 3166-1 alpha-2 standard country codes. You can see how this works at [StructuredFilename.cs](https://github.com/SnowflakePowered/snowflake/blob/d35c4b3c1c7e831fa228090499cdba755064a45d/Snowflake/Romfile/StructuredFilename.cs#L62). If the file name contains GoodTools country codes, Snowflake will know it's a GoodTools ROM. If it's a No-Intro name, it'll know it's a No-Intro, and if its neither, Snowflake will try to parse an ISO 3166 code from the filename. Not only does this tell us what naming format it is, it tells Snowflake, and thus the scraper, what country this ROM is from. If it can't find a code, it warns the scraper that the ROM is improperly named.

The next step is to parse the year. Only GoodTools and TOSEC has date data; years are easy to parse from the date. They always start in 19 or 20, and are 4 numbers or (19/20XX). Snowflake simply looks up any string that starts with 19 or 20 and are 4 numbers inside parentheses or brackets. 

Titles are the easiest to parse. For all 3 formats, the title is simply any part of the file name not in brackets or parentheses. Using regular expressions we can easily get the title. TOSEC uses ending articles, for example, _"The Legend of Zelda"_ becomes _"Legend of Zelda, The"_. This is undone for consistency in all 3 formats, so the scraper only sees _"The Legend of Zelda"_.

Now we have an object that represents all the data from the filename. How do we find out what platform a ROM is?

Every ROM is different &mdash; FileSignatures
--------------------------------------------

Like snowflakes (pun intended), ROMs for different platforms are all different, but in similar ways. ROMs often have their own headers or footers that tell the system how to boot them. We could always look up the full CRC32, but this could be extremely time consuming to hash every byte in the ROM. ROM headers also contain additional data, such as an internal name or game ID that can be used to look up in a database for a more accurate check than a hash.

Instead of using a chainsaw like hashing the file, FileSignatures are specialized scapels for every type of ROM. For example, all Wii games have the magic sequence of bytes [`0x5D, 0x1C, 0x9E, 0xA3`](http://wiibrew.org/wiki/Wii_Disc) at position _0x018_. Now it's easy to find out if an ISO is a Wii game, simply [check](https://github.com/SnowflakePowered/snowflake/blob/master/Snowflake.FileSignatures/Nintendo.Wii.FileSignature.cs#L22) if the file has that data. Because FileSignatures are only run for the file extensions they register for, checking specific bytes like this is extremely fast. 

Systems like the Wii, SNES, and even the Genesis have an internal name and ID inside. By looking at specific offsets, we can even [get this data](https://github.com/SnowflakePowered/snowflake/blob/master/Snowflake.FileSignatures/Nintendo.Wii.FileSignature.cs#L45) for scrapers to use. There are now file signature comparators with varying support for internal names and IDs for [most systems ](https://github.com/SnowflakePowered/snowflake/tree/master/Snowflake.FileSignatures)that Snowflake supports

Of course, there are outliers for bad dumps and some ROMs don't all obey the requirements of their headers. Snowflake tries to guess as best as it can using the file extension if a FileSignature can't find a result, and falls back to a user-specified platform if multiple platforms share the file extension. The best solution here is to not use bad dumps.

Putting it all together &mdash; ScrapeEngine
-------------------------------------------
ScrapeEngine handles putting all this data together. ROM IDs and internal names, as well as the parsed filename is put inside a [`ScrapableInfo`](https://github.com/SnowflakePowered/snowflake/blob/master/Snowflake.API/Scraper/IScrapableInfo.cs) object and passed along to the scraper.

*But which scraper to choose?*

<del>XBMC</del> <ins>Kodi</ins> solves this by having the user choose a scraper beforehand. But games are not TV shows and movies; a source only for MAME games won't work at all when you're trying to find out data for _Super Mario Galaxy_. Likewise, GameTDB is only useful for Wii games and Gamecube games, and won't help at all if you're trying to find data for _Crash Bandicoot_.

Scrapers are already discriminated by which platforms it supports through the plugin-wide `Platforms` property. But some scrapers are more accurate than others, and each one has edge cases where one may beat it out.

ScrapeEngine handles this by sorting each scraper by a base-accuracy value from 0% to 100%, which is set at the discretion of the plugin author. The most accurate scraper plugins are used first, and each result is also assigned an accuracy value, sometimes by how close the scraped title is to the title determined by StructuredFilename (fuzzy), or sometimes whether or not the Game ID matches (absolute). Absolute scrapers such as GameTDB looking up a Wii game ID will always have a base accuracy value of 100%, so they are always run first, while others such as one using TheGamesDB data from a title has a base accuracy of around 85%. This is then averaged with the accuracy of the result. 

Usually if an absolute scraper can't find anything, it'll return no results or results with 0% accuracy, simply because the ROM isn't in it's database (for example, OpenVGDB only has the newest No-Intro dump by CRC32, some usable dumps are not in the database). If the fuzzy search results are within a threshold of about 85%, Snowflake will select that one as the  one most likely to be your game. It's still possible to choose which scraper to use manually, and which result to use manually by asking the scrape engine to use a certain scraper or return all results, but this new algorithm will pick the best match 99% of the time.

Next Up
-------
Smarter scraping is just one of many partial-rewrites of Snowflake. I'm already working on new ways to get controller data and generate emulator configs, and have an article in the pipeline about the new database code that makes that possible. If you have any new ideas on how to improve the scraping system described here, feel free to [post an issue](https://github.com/SnowflakePowered/snowflake/issues/new), and I'll most likely make it happen to make sure Snowflake will have the most accurate scraping possible. I haven't posted new progress logs for a while, and probably won't, but let me know if you prefer this format over the old quick summary of pull requests since the past progress report. Thanks so much for sticking with me all this time, I really appreciate it.
