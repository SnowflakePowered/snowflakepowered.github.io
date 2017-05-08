---
layout: post
title: Snowflake Progress Report - May 2017
---

It's been nearly a year since the last blog, mostly because I've been quite busy with other obligations that I haven't been able to work with Snowflake all that much. However, the last blog was in May of last year, and while working on a bunch of new features I haven't had time to write about them until now, so this 'progress report' is more of a recap of the last half-year of features. IBecause these changes were more under-the-hood, this article will be more of a devlog than a feature preview, and will be talking about the more technical details on how Snowflake is being built.

**If you know how to inject an overlay CEF on top of an arbitrary OpenGL/DirectX surface like the Steam Overlay, [please let me know!](https://github.com/SnowflakePowered/snowflake/issues/235)**

## **Records** &mdash; Multi-file Games and even smarter scraping (Pull Requests [#233](https://github.com/SnowflakePowered/snowflake/pull/233) and [234](https://github.com/SnowflakePowered/snowflake/pull/234))


When it comes to frontends, the traditional approach is one ROM file per game. After all, when you send arguments to an emulator executable, usually you can only pass in one file, be it a `.nes` ROM or a `.cue` for a Playstation rip. However, this quickly becomes unwieldy as Snowflake has to juggle and manage not only ROMs that consist of multiple files (such as any non PBP Playstation rip), but also the multiple resources and save files, as an API is provided for per-game, per-emulator save file separation.

A "*record*" is how Snowflake stores games and files in it's internal database. Each record has a unique `Guid` that can be used to refer to itself through an API, and can contain some required fields (such as `PlatformID` for a game or `FilePath` for a file record), some metadata, and references to other records through it's `Guid`. There are currently two types of records, [game records](https://github.com/SnowflakePowered/snowflake/blob/6a6cb896658dd015917669ed4f4e76e7d4db1fca/src/Snowflake.Framework.Primitives/Records/Game/IGameRecord.cs), and [file records](https://github.com/SnowflakePowered/snowflake/blob/6a6cb896658dd015917669ed4f4e76e7d4db1fca/src/Snowflake.Framework.Primitives/Records/File/IFileRecord.cs); [metadata](https://github.com/SnowflakePowered/snowflake/blob/6a6cb896658dd015917669ed4f4e76e7d4db1fca/src/Snowflake.Framework.Primitives/Records/Metadata/IRecordMetadata.cs) is also stored as a record-like object, but can only have a key and a value, must be associated with another record, and can not be linked to metadata itself.

Through record references, an organized graph can be created for a piece of data.

```json
{  
   "Guid":"bda629d3-2907-453b-8159-e9b17e056c15",
   "PlatformId":"NINTENDO_NES",
   "Title":"Super Mario Bros.",
   "Metadata":{  
      "game_platform":{  
         "Key":"game_platform",
         "Value":"NINTENDO_NES",
         "Guid":"62af9a06-4deb-518a-80e5-fac94d556c5a",
         "Record":"bda629d3-2907-453b-8159-e9b17e056c15"
      },
      "game_title":{  
         "Key":"game_title",
         "Value":"Super Mario Bros.",
         "Guid":"e715a056-fad7-55e1-8401-0aa459f0a5ea",
         "Record":"bda629d3-2907-453b-8159-e9b17e056c15"
      },
      "game_description":{  
         "Key":"game_description",
         "Value":"Super Mario Bros. is a platform video game developed and published by Nintendo for the Nintendo Entertainment System home console.",
         "Guid":"e715a056-fad7-55e1-8401-0aa459f0a5ea",
         "Record":"bda629d3-2907-453b-8159-e9b17e056c15"
      }
   },
   "Files":[  
      {  
         "Guid":"9fcd9379-c5a3-4607-80dd-4deacc38750a",
         "MimeType":"application/x-romfile-nes-ines",
         "FilePath":"C:\\roms\\nes\\smb1.nes",
         "Record":"bda629d3-2907-453b-8159-e9b17e056c15",
         "Metadata":{  
            "file_linkedrecord":{  
               "Key":"file_linkedrecord",
               "Value":"bda629d3-2907-453b-8159-e9b17e056c15",
               "Guid":"8d282347-d73a-542d-a88a-dca799bad463",
               "Record":"9fcd9379-c5a3-4607-80dd-4deacc38750a"
            }
         }
      },
      {  
         "Guid":"fdc2dea1-3821-4fd5-b7f0-f8de7ef6cc62",
         "MimeType":"image/jpeg",
         "FilePath":"C:\\users\\appdata\\snowflake\\imagecache\\100_9957466a-0307-4f9d-9625-509b9ee24445.jpg",
         "Record":"bda629d3-2907-453b-8159-e9b17e056c15",
         "Metadata":{  
            "file_linkedrecord":{  
               "Key":"file_linkedrecord",
               "Value":"bda629d3-2907-453b-8159-e9b17e056c15",
               "Guid":"55a4a6ff-d563-4a6e-9451-16bee9329bb8",
               "Record":"fdc2dea1-3821-4fd5-b7f0-f8de7ef6cc62"
            },
            "image_type":{  
               "Key":"image_type",
               "Value":"media_boxart_front",
               "Guid":"3f8decc0-9cd4-487f-95ec-c88fa67c84e2",
               "Record":"fdc2dea1-3821-4fd5-b7f0-f8de7ef6cc62"
            },
            "image_cache_id":{  
               "Key":"image_cache_id",
               "Value":"9957466a-0307-4f9d-9625-509b9ee24445",
               "Guid":"b7ff2512-5b7f-411a-a3b4-25c94bc6e8da",
               "Record":"fdc2dea1-3821-4fd5-b7f0-f8de7ef6cc62"
            }
         }
      }
   ]
}
```

In this example, this JSON graph of a game contains all the information needed to display, and launch "*Super Mario Bros.*", including not only the ROM file, but relevant metadata and media, in the form of Records. What might not be clear are the fields labeled `Record` that exist in each file record or metadata, is a reference to to the record the data is associated with. For example, both file records link to the game record with the Guid `bda629d3-2907-453b-8159-e9b17e056c15`. This circular reference of sorts reflects how the data is stored in the database in a flat structure, and allows for the client to traverse up the tree given an identified metadata or file. 

Records allow Snowflake to take a higan-like approach to games. A game can have multiple files associated with it, including images, marquees, save files, and videos. A file or a game can have some metadata attached to it, such as the date of creation for a save file, or how well it runs on a certain emulator. Other use cases including having multiple ROM formats being stored under the same game; the emulator choosed to launch the game would simply discriminate which one it can run by `stone` mimetypes. ROM formats such as DOS and BIN/CUE/WAV could have every file stored in the database and used to verify the integrity of the game, The possibilities are endless, and this system of Records and metadata allows for a consistent and central interface for emulators and plugins to utilise. Already, scrapers, specifically media scrapers, are emitting results in the form of `FileRecords` of cached images; this greatly simplifies scraping for images and other media resources for a game, as well as allowing a great deal of flexibility than the older restrictive system before.

## **DynamicProxy configuration API** (Pull Request [#239](https://github.com/SnowflakePowered/snowflake/pull/239)) &mdash; Faster configuration generation.

This was the third rewrite of Snowflake's on-the-fly configuration generation API, and while API-wise remained similar, allowed for some unique tricks to make emulator wrappers not only easier to write but more performant, minimizing usage of reflection by caching generated configurations. It also allows for better IntelliSense when working with configuration templates and reduces the boilerplate needed by nearly half.  

Templates were originally concrete classes that derived from a `ConfigurationSection` base class. Defining an option looked like this:
```csharp
[ConfigurationOption("video_fullscreen", DisplayName = "Enable Fullscreen", Simple = true)]
public bool VideoFullscreen { get; set; } = false;
```

With this setup, the value of the configuration was stored as property of a class with no special faculties to access it. Therefore, to convert a template into a valid configuration file, it would be reflected over multiple times, without any caching, in order to find not only the value of the option, but the metadata required to serialize it, every time a game launched. Since configurations are often very large and complex, consisting of multiple classes that each have to be reflected over, this was very slow and costly, taking almost 10 seconds to launch a RetroArch core. Moreover, by the time it gets to the serializer, most of the type information has been erased; the serializer only sees a bunch of `ConfigurationSection` base classes, with no way to know what it contains except through Reflection. 

If we were able to have more control over how the options are stored, we would not need to rely on Reflection in order to view the values and their metadata. However, storing them in a plain Dictionary, while an order of magnitude faster, was simply unacceptable when we lose the strong typing and IntelliSense when working with these configurations. Third-party cached Reflection libraries helped such as Fasterflect, but they weren't fast enough and did not provide enough flexibility. 

[DynamicProxy](http://www.castleproject.org/projects/dynamicproxy/) is a library that allows classes to be generated at runtime from an interface, with their calls intercepted in a certain manner. This was perfect for Snowflake's use case, and allowed for a huge improvement in the Configuration API. With the new API, defining an option now looks like this:

```csharp
[ConfigurationOption("video_fullscreen", false, DisplayName = "Enable Fullscreen", Simple = true)]
bool VideoFullscreen { get; set; }
```

Instead of being a property of a class, an option now lives as property of a generic interface of `IConfigurationSection<T>`, where `T` is itself. In other words, [every section implements `IConfigurationSection<T>` of itself.](https://github.com/SnowflakePowered/snowflake/blob/master/src/Snowflake.Plugin.Emulators.RetroArch/Configuration/VideoConfiguration.cs). This monadic pattern is seen in classes such as `IEquatable<T>` or `IComparable<T>`. Because C# has first-class properties, this allows us to use DynamicProxy to implement an instance of `IConfigurationSection<T>`, and intercept every time a property is accessed: Snowflake implements the getter and setter at runtime rather than the plugin author.

Using this interception behaviour allows Snowflake to use a Dictionary to store the value of each configuration option, as well as cache the metadata stored in the attributes when the class is initiated at runtime. DynamicProxy caches the generated implementation, so the reflection is done only once to reflect over the attributes in the interface, rather than every time a configuration needs to be generated.

This approach is not only more performant than reflecting over class members, but allows us to keep many of the benefits, such as strong-typing, while allowing us to drop to weaker typing when required by accessing the backing dictionary, exposed as immutable and readonly to the API consumer. Options are modified just as you would any other class, but when it comes time to serialize, the serializer simply accesss the backing dictionary and loops through the dictionary to serialize the configuration option. 

This concept of intercepting an interface by generating a dictionary-backed implementation at runtime is used frequently throughout the configuration API. `IConfigurationSection<T>`, representing a section of configuration, is grouped into a `IConfigurationCollection<T>`, [representing a collection of configuration sections](https://github.com/SnowflakePowered/snowflake/blob/6a6cb896658dd015917669ed4f4e76e7d4db1fca/src/Snowflake.Plugin.Emulators.RetroArch/RetroArchConfiguration.cs), allowing access to the backing values as a dictionary of dictionaries rather than having to reflect through each member, and reflect again. There are some other changes in the API, such as the `ConfigurationFileAttribute`, allowing sections to be tagged with a tag, which could be specified to output to a specific file by the serializer. The input configuration API is similarily implemented with an interceptor and a dynamic proxy.

Personally, I was completely blown away when I had this working, but there are still some improvements to be had. Due to the nature of the circular reference in the interface declaration, there is a self-reference in every configuration section, allowing for things like so:

```csharp
IConfigurationSection<MyConfiguration> myConfiguration = new ConfigurationSection<MyConfiguration>();
myConfiguration.Configuration.SomeValue = true; 
```

In this case, `myConfiguration.Configuration` is an instance of `MyConfiguration` that was generated by the proxy, but since `MyConfiguration` must implement `IConfigurationSection<MyConfiguration>`...

```csharp
// This also works.
myConfiguration.Configuration.Configuration.Configuration.Configuration.SomeValue = true; 
```

This behaviour is actually intended and required interceptors that [do nothing but handle this circular reference](https://github.com/SnowflakePowered/snowflake/blob/6a6cb896658dd015917669ed4f4e76e7d4db1fca/src/Snowflake.Framework/Configuration/Interceptors/ConfigurationCircularInterceptor.cs). If this circular interceptor wasn't here, then `myConfiguration.Configuration.Configuration` would be a null reference exception. While the circular interceptor is a rather elegant solution, it still seems a bit hacky, and I hope to clean it up. To be fair, all the interceptors are a bit hacky with switch statements and whatnot, so I hope to utilise some of the new pattern matching features in C#7 soon.

Another optimization that should be possible soon is that DyanmicProxy allows for saving the generated implementations to a DLL. Currently, the implementations are generated once per application lifetime, so the initial start of a certain emulator will be slow (in comparison to the next usage), every time Snowflake is restarted. However, this feature is currently unavailable for .NET Core, which leads us to the next topic of this devlog.


## **Porting to .NET Standard** (Pull Request [#241](https://github.com/SnowflakePowered/snowflake/pull/241)) &mdash; First-class cross-platform support*

Snowflake was originally written on the .NET Framework. Fast forward a few years later, [.NET Standard](https://docs.microsoft.com/en-us/dotnet/articles/standard/library) matured enough for Snowflake to compile and pass non-input unit tests. Snowflake now runs completely on .NET Core, and no longer supports running on top of .NET Framework. It runs on the same runtime library on both Windows, OSX and Linux, thus has first-class cross-platform support.

In terms of the code, the only thing that had to be rewritten was the Plugin loading. It's still a bit buggy, and in fact migrating to .NET Standard brings with it some additional issues regarding dependencies and packaging, so it will probably have to be rewritten again, but it suffices for now. A few Windows-isms were also removed in the code to make it truly cross-platform.

However, since migrating over to the new project type had to be done manually, this provided a chance to reorganize the solution to something that makes a bit more sense. 

First level `Snowflake.*` namespaces generally became `Snowflake.Framework` namespaces, with `Snowflake.API` becoming `Snowflake.Framework.Primitives`, and `Snowflake` becoming `Snowflake.Framework`. Certain plugins became `Snowflake.Support` plugins, these plugins usually are required for full functionality but don't make as much sense to include into `Snowflake.Framework`. 

Finally, auxillary tools like `Snowball` and `Shiragame` were moved to the `Snowflake.Tooling` namespace. This namespace actually took the biggest hit from the migration. `snowball.exe` is all sorts of broken and requires a complete rewrite, and since `Microsoft.Data.Sqlite`, the replacement for `System.Data.SQLite` in .NET Core does not support in-memory databases, building a new Shiragame database with the .NET Core version will take much longer as each one of the thousands of transactions must be written to disk. However the tooling is not necessary to run Snowflake and were in need of some cleanup anyways.

The one caveat is of course that both the bootstrap, emulator plugins, and input manager have to be rewritten for each operatinng system. Currently, there is no support for launching emulators on a non-Windows operating system, and while the input manager does build on Linux, since it uses SharpDX, which abstracts over DirectInput, obviously it will not work on any other OS but Windows. I will have to look into how Linux and OSX handle input, specifically how input devices are enumerated and exposed to applications, before any real support can come to these operating systems.

On the bright side though, plugin loading, scraping, config generation, and the game database all work fine on Linux. Everything works except input, which of course is the most complicated and low-level part of Snowflake. That will take some time to figure out, along side the plugin system rework, also necessary because of the switch from .NET Framework. 

## **Remoting and React-based UI** &mdash; Javascript interop and client API

One of Snowflake's main features is the client/server model; the server, or "framework", handles managing the game database, launching games, etc., while the client, or "theme", handles the actual "emulator frontend" part by displaying games to the user in a friendly interface. Up till now, the focus has been on the framework API, but recently I've been working on the client Javascript API and a preliminary theme. 

What used to be `Snowflake.StandardAjax` has been reworked into a plugin, `Snowflake.Support.Remoting`. The old RPC-style calls are being rewritten in a more REST-like architecture with resources being accessible by their URL and Verbs instead of being an RPC-call. Doing it this way allows the protocol to be abstracted; while currently I've reverted back to using HTTP, it can easily be adapted to protocols such as WebSocket later on. The method calls are written to return C# objects instead of `JsonResponse`, and are formatted as Json by the server as late as possible. This also allows `Snowflake.Support.Remoting` to be used as a general client API for C# applications, since the responses are the same types of objects as the framework uses in it's internal API. 

While these responses can be used by their own as raw JSON, there are [Typescript bindings](https://github.com/SnowflakePowered/snowflake/tree/remoting/js/snowflake-remoting) that make the API easier to use. The bindings also enforce immutability in the returned objects using `seamless-immutable`. Since they're written in strict Typescript, any client witten in Typescript will have access to the type definitions in the API.

As for the theme proper, past attempts involved Web Components and [Polymer](https://github.com/SnowflakePowered/theme-paper-snowflake), but this never panned out very well. Instead, I've been working on a [new UI](https://github.com/SnowflakePowered/snowflake/tree/remoting/js/snowflake-react-boilerplate) using React.

![Game Cards](http://i.imgur.com/EO1mwDO.gif)

Using `material-ui`, I'm starting to build the individual components that make up the UI, you can see some of the initial work in this [storybook](http://snowflakepowe.red/storybook-static). Eventually, the `material-ui` parts will be refactored out and will be left with a basic boilerplate that themes can use to get started. I haven't been able to integrate Typescript very well with React yet, nor do I want to use Flowtype, but I'm hoping to gradually fold type-checking into the codebase soon. 

The focus for the next few months will be completing the UI and alongside, the Remoting API. At that point, an Alpha preview release will be possible, *tentatively for Summer 2017*. People have asked me about launching a Patreon, and while I'm not opposed to the idea, I have qualms about crowdfunding before I have a usable alpha, so I will probably sort this out once I can release the alpha. After the alpha, the next steps will be to sort out the plugin loading on .NET Core before writing adapters for more emulators. Any Patreon funds would go towards writing and supporting these adapters, as well as cloud services and writing quality API documentation on Snowflake's plugin and remoting API. In the future, I hope to be writing more frequently about features; I used to do it once a month but many of the features in this article took multiple months to realise. For up-to-the-minute updates on new features, [star the GitHub repository](https://github.com/SnowflakePowered/snowflake), or even PM me if you're interested about something, I'd be glad to give you a rundown on what I've been up to! 
