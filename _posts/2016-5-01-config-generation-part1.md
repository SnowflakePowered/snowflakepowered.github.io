---
layout: post
title: Feature Preview - On-the-fly Configuration Generation Part 1.
---



# Feature Preview — On-the-fly Configuration Generation Part 1.



This is part 1 of a multi-part series on the concept of "*configuration generation*", Snowflake's solution for integrating with emulators. This article talks about features soon-to-be merged into the main branch, but are not yet available, when the features are merged, this article will be updated accordingly.

Emulation frontends generally follow one of two approaches. Traditional frontends such as *LaunchBox* or *EmulationStation* among countless others simply launch an emulator executable, passing along command line parameters to the proper position, leaving everything to the actual application that is launched. Core-based frontends, such as *RetroArch* or *OpenEmu* implement a standard API platform that an emulator can target, while the frontend handles rendering, input, and a host of other tasks. Both approaches have their advantages and disadvantages. Traditional frontends have a low-setup cost, but sacrifice control over the emulator. A core-based frontend requires emulators to be specifically rewritten in a certain way to target their API, (usually *libRetro*), and as a result, some emulators are not supported. 

Snowflake takes somewhat of a middle ground using a technique called *configuration generation*. It is exactly what it sounds like; by generating a set of configuration files for an emulator, Snowflake can more-or-less expose the options of the emulator from the frontend, without resorting to more invasive options such as the solution *RetroArch* provides. This approach comes with it's own disadvantages: many emulators don't have well documented configuration options, and the set up cost is higher than traditional frontends that only require a config file. However, emulators do not have to be re-compiled especially for the platform, and support is universal, as long as the emulator reads it's options from a writable file or registry entry (looking at you ePSXe.)

## Comparison chart

Here is a comparison chart of all three approaches to how frontends handle emulators.

|                          | Traditional Frontends                    | Snowflake                                | Core-based Frontends                     |
| ------------------------ | ---------------------------------------- | ---------------------------------------- | ---------------------------------------- |
| **Compatibility**        | Traditional frontends have compatibility with any emulator as long as they take in a ROM name as a command-line parameter. | Each emulator must have an "adapter" plugin in order for Snowflake to understand how to generate configuration and launch the game. | Each emulator must be non-trivially "re-targeted" to the frontend API. As a result, not all emulators can be supported, especially closed-sourced emulators and emulators that are tied to an OS. |
| **Setup burden**         | The burden of setup is on the end-user who must manually configure launch paths and set up the folder structure for emulator applications. | The burden of setup is on the developer of the adapter, which handles configuration generation. | The burden of setup is on the developer, who must rework the emulation logic of an emulator and plug it into the frontend API. |
| **Glue-code complexity** | Usually simple XML or INI configuration files. | The developer must study the configuration format of the emulator and determine how to generate a valid configuration. | The developer must study the inner workings of the emulator code, and be able to hook up the respective functions to the frontend API. |
| **Integration**          | Almost no integration. Emulators must be set up individually, and there are no facilities to do so inside the frontend. | Surface level integration. User-friendly options can be changed by the user inside the frontend, and event hooks provide access to the emulator lifecycle, but a lot of control is lost when the emulator executable runs to the level of traditional frontends. Some ways to mitigate this would be in-game overlays that hook graphic APIs. | Complete integration. The emulation runs inside the frontend process, which has complete control over the output and input of the emulation running. |
| **Input Handling**       | None, the emulator must have been set up previously to handle input. | Input configuration is generated and passed on to the emulator before launch. | Input is handled completely by the frontend. |
| **Ease of Use**          | Requires users to set up configuration files. | Ready to go once ROMs are added; "adapters" take care of configuration. | Varies; some frontends expose too many options to users. |

## Why configuration generation?

Configuration generation takes on some disadvantages of both alternate approaches. Non-trivial glue code must be written to fully take advantage of the emulator, and while the use of in-game overlays can mitigate some of the issues, it can never reach the full integration of core-based frontends such as *OpenEmu*. However, this approach allows Snowflake to be a lot more flexible. Emulators such as *Dolphin*, *PCSX2* and *Cemu* fall within the realm of possibility with surface-level integration, a huge improvement from the absolute zero level of integration provided by traditional frontends. The barrier to writing an adapter plugin for Snowflake is much lower than writing an emulator core, all that is needed is knowledge of an emulator's configuration options, rather than it's inner workings. Essentially, Snowflake takes what the user used to do individually for each emulator in a traditional approach, and automates it; underneath Snowflake is still simply calling an executable with command-line arguments. The emulator application can be updated separately without having to be re-adapted into a core, as long as configuration options are not drastically different. While very much a "good enough" solution, the tradeoffs made for increased maintainability over core-based frontends, and ease-of-use over traditional frontends make configuration generation a flexible and useful approach.

## How it works

An emulator application (we call it an "assembly" as well)  usually has some way of storing configuration, usually in a machine-readable format. Snowflake requires each supported application to write what is called a "Handler", or adapter plugin that tells it which options are available, and how to format it. 

Defining an option looks like this in Snowflake.

```c#
[ConfigurationOption("video_fullscreen", DisplayName = "Enable Fullscreen", Simple = true)]
public bool VideoFullscreen { get; set; } = false;
```

Even if you can't read code, the intent is obvious at first glance. This piece of code tells Snowflake that there is an option called `"video_fullscreen"`, and that the default value is `false`. The `DisplayName` and `Simple` options tell Snowflake that this option is user-friendly, and gives it a user-friendly name.  `bool` means it can be either `true` or `false`, and it is stored internally as `VideoFullscreen` (not `video_fullscreen`), before being converted into configuration format, which looks like this:

```c
video_fullscreen = "false"
```

which is then read in by, in this case *RetroArch* running some core (Snowflake wraps this behaviour using these plugins), and understands that the user wants to play in Windowed mode.

Options are grouped into sections, which are grouped into `ConfigurationCollection`s, which represent a single potential configuration file within Snowflake, containing information about the filename and location.

The conversion of this information into one line can be handled differently per-emulator by writing a custom `ConfigurationSerializer`, which takes configuration options and transforms them into valid configuration. When the game is launched, the information from all these `ConfigurationOption`s are converted into a single `retroarch.cfg`. 

For example, this following code which represents a section of the RetroArch video configuration

```csharp
[ConfigurationOption("video_fullscreen", DisplayName = "Enable Fullscreen", Simple = true)]
public bool VideoFullscreen { get; set; } = false;

[ConfigurationOption("video_driver", DisplayName = "Video Driver", Simple = true)]
public VideoDriver VideoDriver { get; set; } = VideoDriver.Direct3D;

[ConfigurationOption("video_scale", DisplayName = "Window Scale", Simple = true)]
public double VideoScale { get; set; } = 3;
```


can be represented like this following mockup: (mess with it!)


<p data-height="266" data-theme-id="dark" data-slug-hash="bpQgyz" data-default-tab="html,result" data-user="RonnChyran" data-embed-version="2" class="codepen">  </p>
<script async="async" src="//assets.codepen.io/assets/embed/ei.js"> </script>


allowing the interface to easily expose emulator options to the user, and generate valid configuration.

## Part 2

Stay tuned for a part two going in-depth the solution to generating input configuration on-the-fly. Input configuration is a lot more complicated and involved than general configuration, because it can only be determined right before a game starts. Snowflake's unique solution involves multiple parts interacting with the operating system to get devices, and "definition" files to tell Snowflake how to turn this information into valid configuration!.

