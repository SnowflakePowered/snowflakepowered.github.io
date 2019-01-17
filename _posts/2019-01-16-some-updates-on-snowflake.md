---
layout: post
title: Some updates on Snowflake
---

It's been quite a while since my last post on this blog, and until recently it's been quite a while since I last pushed any updates to Snowflake itself. I felt that some updates are in order for those that have been following this project for so long. This is not necessarily a progress report, but more of a summary of what has happened since last year, and an overview of the direction the project is going. This is going to be a very long blog post, and may be the last one for perhaps quite a while.

First off, **Snowflake isn't dead**. I remain committed to working on this project until at least a feature complete 1.0 version, with a working user interface, that I am able to use myself on a daily basis. Every project I've worked on I've done so primarily for my own usage, so I believe that dogfooding (using your own product) is incredibly important to deliver a quality product.

As for why things are taking so long, there are a variety of reasons. When I started this project five years ago  (*christ, it's been five years!*), I underestimated what was required to create a frontend in a general way such as Snowflake, and that was during a time when I had much more free time to spend on side-projects such as this. It turns out that an emulator frontend is a much bigger problem space than would be expected especially one as ambitious as what Snowflake aims to be. Some may chalk this up to feature creep, but in reality Snowflake has always aimed to have, and has had these core features.
  
  * Support for **all types of emulated games** 
  * Being able to generate configuration for **any executable emulator**, gaining more control over usual command-line or AHK based frontends
  * Support for accurate scraping from any source
  * A nice user interface 
  * An API for that nice interface, so that it can be written in any language
  * Can be extended through a plugin system
  
Snowflake has always had these features from a very early point, but the problem was each of these features are difficult to implement so that they work together well, and are easy to use. There has been three iterations of the plugin system, three of the games library, et cetera. I hope at the end however, that everything will be worth it. 

Now with that out of the way, let's talk about some actual changes made recently. 

A warning beforehand, due to the nature of these changes there hasn't been many new features to talk about. Instead, most of this blog post will be technical and include technical jargon, mostly without anything to do with emulation. 

## Support GraphQL as the canonical remoting solution, using Kestrel as the default web server.

GraphQL support was introduced nearly a year ago, replacing ad-hoc REST API and WebSocket solutions with a much better query language. Back then, I had intended to have Snowflake support a variety of remoting protocols, including binary formats such as Protobuf, and the return of the REST API in the future.

However, I now realize that since Snowflake was built as a collection of composable parts, the remoting API is too far integrated to have multiple implementations. The procedure for which to perform an action such as executing a game, or adding one to the game library, belongs solely as part of the remoting API and nowhere else as these actions tie the parts of Snowflake together.

As such, Snowflake now **only supports** GraphQL as the only way to do actions. Using Kestrel, GraphQL queries can now be made over WebSockets using the [Apollo WebSocket Protocol](https://github.com/apollographql/subscriptions-transport-ws) rather than HTTP for speedy queries.

The API to register new APIs to the remote has also been improved.

C# interfaces could also presumably call the remoting implementations directly, but it is still recommended to marshal access through GraphQL to maintain guarantees of consistency between the server backend.

## A new game library paradigm

The easiest change to talk about here is that the game library now uses EntityFramework Core, rather than handwritten SQL. It's not completely ready yet since Migrations haven't been implemented however. If this means absolutely nothing to you, don't fret. I promise the other changes will be more interesting.

Snowflake's Game Library used to look something like this.

![old](https://i.imgur.com/tSszHxA.png)

You had a **Game record** in the database, and a bunch of **File records** that linked back to the Game record. Each of those records could own some metadata.

With the recent changes to Snowflake, the game library looks more like this.

![new](https://i.imgur.com/5XWizID.png)

Looks a bit more complicated, but addresses some concerns with the old architecture.

The main concern was that without a **Game record** having been available before, it was impossible to create an associated File record. At first this doesn't sound like a problem, but what if you consider images and media that were gathered during scraping, before all the information for a Game record to be created was available yet? As well, file records mandate a known mimetype. While this is often useful, we don't necessarily need to know mimetypes for *every file type*.

Another concern was that simply keeping track of the file's path on a disk is extremely fragile, and Snowflake would have no way of knowing whether or not a file moved. Not only that, it was a leaky abstraction, exposing the file system to any user interfaces. This increases the chance of bugs where file paths and behaviour differ on different operating systems.

The new design **decouples** files from **games**, while still allowing associations between files and games as well as both files and games being able to have metadata.

With the new *model* API, when a *game* is first created, the only required data is its platform. An empty record is created, as well as a folder where all game data will be relative to. 

Files that are added to the *game folder* through the Filesytem API will be tracked using a unique ID and a folder-local manifest. Games can access this file system relative to itself, for example with a game with UUID `12345-1203-...`, the path `/program/game.rom` would map to `%appdata%/.snowflake/game/12345-1203-.../program/game.rom` by default. These files are enumerable relative to an instance of Game.

However, simply adding a file to the game folder does not a File record make. Instead, File records must be **explicitly registered with the game** using a known mimetype. Only then will files be able to associate with a mimetype. File records do not use the game path to track the file, but instead using the the folder-local UUID of the recorded file. Hence, as long as files are moved within the Filesystem API which transfers UUIDs, metadata attached to a file will persist, **even across game boundaries**.  Not only that, if a file is deleted, it will no longer be enumerable, and thus, the File records, although persisting in the database, will no longer be accessible, providing **automatic cleanup of File records**.

Games don't simply contain File records now. Instead each instance of a game acts as its own [service container and can contain services](https://github.com/SnowflakePowered/snowflake/tree/master/src/Snowflake.Framework/Model/Game/LibraryExtensions) by registering with the Game library. The system of files mentioned above is implemented this way, but now Configurations are accessible [**through a Game instance**](https://github.com/SnowflakePowered/snowflake/blob/master/src/Snowflake.Framework.Tests/Model/GameLibraryIntegrationTests.cs#L35), as well as any arbitrary extension.

Game-related data used to be accessed through multiple disparate databases by passing around UUIDs. With the new API, games are now extensible and game-related data can be accessed local to the game. These changes make working with the GraphQL remoting API as well as writing emulator plugins much more ergonomic.

## .NET Core Plugin SDK

While this doesn't have anything to do with Snowflake's code itself, Snowflake now defines it's own custom .NET SDK for a variety of purposes; mainly to ensure consistency of dependencies within plugins. With this, plugins can be packaged much more compactly by only packing direct dependencies that Snowflake supplies during runtime.

As part of the SDK, the following third-party dependencies are exposed to consumers

* System.Collections.Immutable
* System.Dynamic.Runtime
* System.Linq.Parallel
* Microsoft.Data.Sqlite
* Microsoft.AspNetCore.Server.Kestrel
* Newtonsoft.Json
* GraphQL
* Zio

To use the SDK, specify this in the `.csproj`

```xml
<Project Sdk="Snowflake.Framework.Sdk/1.0.0">
```

and put the Snowflake MyGet Package Source in the `nuget.config` file

```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
    <packageSources>
        <add key="myget-snowflake" value="https://www.myget.org/F/snowflake-nightly/api/v3/index.json" />
    </packageSources>
</configuration>
```

Snowflake now also supports `dotnet new` for creating new plugin projects.

```bash
$ dotnet new -i Snowflake.Templates.AssemblyModule
$ dotnet new snowflake-mod --author "John Doe"
```

## Configuration Changes
In order to facilitate easier access to configuration values, values in a `ConfigurationCollection` instance are now stored in a `ConfigurationValueCollection` instance. This instance can have a primary key across all values and can be saved directly to the database, as well as allowing easier flat serialization to GraphQL rather than a deeply-nested object graph.

This change allows us to implement some additional niceties in the future when working with Configurations, particularly when writing serializers and during serialization. I hope to move towards a full compilation model, where individual `ConfigurationValues` are converted to some form of abstract syntax tree containing type information, the value of the configuration, and the configuration key, which can be finally converted to text or some other format according to *configuration targets* as specified by the emulator plugin; as would traditionally be done all in one step by a `ConfigurationSerializer`. 


## Things in the pipeline &mdash; Installers, Async Scrapers, and Emulation changes

The rewrite of the Game library, in tandem with the Filesystem API that compliments it, was actually done in preparation for **installers**, which is the new way in which games (that is, the actual ROMs and ISO files), will be added to the game library.

We're beginning to see emulators for seventh generation and later consoles that are more PC-like than ever and require games to be first *installed*, or *unpacked*, and often contain multiple files. In such cases, the usual paradigm of having one file per game falls apart.

Although Snowflake has had support for multi-file games for a while, there was no way to apply transformations to files. If a game needed to be extracted, it must be extracted or installed to the user, and Snowflake must somehow discover those files: just requiring the user to extract the files manually is a failure of user experience.

The goal of installers is to encapsulate transformations over ROM files. This role was previously intended for `FileRecord` scrape traversers, but that mechanism was much too general to act efficiently on a restricted problem space &mdash; that of game-related data. 

Installers will also be responsible for determining the mimetype for a given file as well as applying transformations. The most basic transformation would be to copy a file to a game's folder. More complex transformations, such as installing a PS3 `.pkg` package, unzipping a file, can also be supported. Since installers run before scrapers, they can also seed data by applying metadata to a file record.

An *Installer* produces an asynchronous list of *InstallTask*, which, rather than executing the transformations, describes the transformations. Only at the very end are transformations applied after yielding the complete list of tasks to be done. Since tasks are asynchronous, an installer can await a future task to compute the next task in the chain. 

**Asynchronous Scrapers** are the next step after implementing the *Installer API*. This will look very similar to the current *tree-scraping* implementation that allows metadata to be added in a tree conditional to its siblings, parent, and children. However, since installers and the filesystem API replace the need for creating FileRecords for all types of files, traversers will be removed. Async scrapers will also take advantage of `IAsyncEnumerable` to remove much of the boilerplate required when working with the Scraping API. However, I have yet to sketch out the exact details of what will change besides the fact the current tree paradigm will remain intact. 

Finally, perhaps the least ambitious change, will be a way to reconcile the current Emulator API with the new game library. Consolidating data under a game instance allows for new possibilities, and the installer API allows for installers to run **right before launch time**. Such possibilities could include an emulator plugin converting a game on the fly for a particular required format, or managing save files from different emulators relative to a game folder. 
