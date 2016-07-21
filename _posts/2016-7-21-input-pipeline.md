---
layout: post
title: The Input Pipeline — A closer look at Snowflake's input device management.
---

*This is kind of a pseudo-devlog in which I go into slight technical detail about one of Snowflake's multiple APIs. I've written it in such a way as to not require much if any technical knowledge, however, I won't blame you if you find this wall of text a bit boring. You can also consider this the part two of the configuration generation article.* 

**As of the time of writing, this article talks about upcoming API features in PR [#229](https://github.com/SnowflakePowered/snowflake/pull/229), which has yet to be merged. If you are interested, please leave your comments on the API design in the Pull Request discussion. PR #229 is one of the largest rewrites in Snowflake before 1.0 and will be the final configuration and input API.**

Unlike your traditional frontend, [Snowflake takes a hybrid approach](http://snowflakepowe.red/config-generation-part1/) when it comes to working with different emulators. Because of the varied nature of video game controllers and input devices, the input handling pipeline is one of the most complex and involved subsystems in Snowflake, having undergone several rewrites and redesigns.

Handling input is a multifaceted problem that considers physical devices and their use cases, the different ways emulators accept input configuration, how the controller is laid out on the emulated console, as well as special consideration for non-conventional devices such as the Wii Remote and the Nintendo DS Touchscreen. With all this in mind, Snowflake also supports different profiles for every controller, so you can re-map your input method any way you want. Let's explore how all the puzzle pieces fit together into a complex but elegant and powerful solution .

## Controller Layouts

Internally, Snowflake refers to physical video game controller devices as simply '*devices*', while '*controllers*' refer to the layout of the emulated controller. These such '*controller layouts*' form the basis of Snowflake's input handling. Inspired by libretro's concept of a *RetroPad*, controller layouts are as well an abstraction interface across input devices. 

However, unlike the *RetroPad*, controller layouts are defined as data by the [Stone specification](https://github.com/SnowflakePowered/stone), which seeks to document emulated platforms and their controllers in a machine-readable format, is validated by a data schema. It is also quite a bit more expansive than RetroPad, aiming more to be able to encapsulate all possible gamepad configurations, rather than being a standard gamepad that emulators can be controlled by. Controller layouts can have any and all of the buttons, analogs, and triggers; or '*controller elements*' that allow it to describe [most, if not all controllers](https://github.com/SnowflakePowered/stone/blob/master/spec/Controllers.md) available on the market today.

Controller layouts consist of a list of controller elements, as well as a label for each element to define its layout, and a list of supported platforms. With Snowflake, a layout is used for both emulated controllers (from Stone), and for real physical devices. 


## Device Enumeration

On Windows, *DirectInput*, part of the DirectX suite, supports a multitude of varied devices, ranging from your traditional dual-analog gamepad to more esoteric devices like the Wii Remote. Whether DirectInput can translate these inputs from these devices is irrelevant, the fact that it can see them and know that they exist is enough. 

Using plugins called *InputEnumerators*, Snowflake takes advantage of the fact that every unique device has a special ID that tells DirectInput what device it is, and each *InputEnumerator* exposes a certain device to Snowflake. Along with basic information such as the name of the device, and the gamepad index (i.e. Player 1, Player 2, etc.), the *InputEnumerator* provides a predefined controller layout for the type of device exposed, as well as additional metadata such as supported platforms (for example, restricting access to real Wii Remote devices only to Wii emulators). This matching of the device to controller layout is central to how inputs are handled in Snowflake, allowing default mappings to be inferred based on available buttons and analog sticks, without needing to probe too deeply into the operating system. As a downside to this more abstracted, defined approach, an enumerator plugin must be written for each different input device. As of writing, Snowflake supports any conventional XInput device (including wrappers), the XBOX 360 controller in DirectInput mode, the keyboard (handled as a special case), and Wii Remote enumeration. 

## Mapping the Layout to the Device

Before being converted to a certain emulator's configuration format, Snowflake stores it's own internal mappings from the emulated controller layout to the device's layout. Such mappings can be customized by the user, for example, mapping the emulated controller's `ButtonA` to the device's `ButtonB` to swap A and B buttons. Because we know the layouts of both the target, emulated controller, and the real physical device, inferring default is simply a matter of comparing layouts to map A to A, B to B, etc. Snowflake can then store these mappings in a simple database, including multiple profiles, for each combination of emulated controller to the physical device.

### Handling Keyboard Layouts

Stone does not specify a method for handling keyboard layouts beyond the `Keyboard` controller element. However, the keyboard is an important part of emulators on the PC. Snowflake provides it's own keyboard abstraction with 93 supported keys, the layout of a standard QWERTY keyboard with 10-keypad, excluding certain auxiliary function keys. The keyboard device is enumerated with a single controller element of `Keyboard`, the intended use of this element in accordance with the Stone specification. 

Internally, mapped pairs of Layout to Device are thus stored with a Controller Element value of `Keyboard`, and a Keyboard Key value with the corresponding keyboard key. Reciprocally, mapped pairs that do not involve a keyboard have a Keyboard Key value of `KeyNone`. Pointer devices such as mice are handled as normal controller elements, without a keyboard key.

For inferring default values, an internal dictionary is kept that maps keyboard keys to default controller elements.

## Converting Controller Elements to Emulator Configuration

As of yet, this system seems ridiculously overengineered compared to even input handling in games. However, it becomes clear why it has been designed in such a way, once we consider converting our internal mappings of Stone controller layout to the device layout, into valid input configuration.

Snowflake has to support multiple emulators and their configuration formats, which can vary wildly. We can now use the same controller elements to refer to both the configuration option in the emulator, as well as what a certain element, for a certain device, serializes to when converted to the emulator's native input configuration format.

Emulator adapters, in order to successfully serialize input, must have 2 things:

1. An input configuration template that implements `IInputTemplate`
2. An input mapping that maps the device element keys, to the native input configuration format.

An input configuration template, such as the one for [RetroArch's adapter plugin](https://github.com/RonnChyran/snowflake/blob/new-emulator-api/Emulators/Snowflake.Plugin.EmulatorAdapter.RetroArch/Input/RetroPadTemplate.InputOptions.cs), is implemented as a bunch of `IMappedControllerElement` properties, called Input Options. When it's time to start the emulator, a set of input mappings for each player is sent to the configuration serializer, which generates the configuration. The serializer does the following for every player:

1. Find the property that is marked for the type of device (keyboard or gamepad), and the emulated controller element (for example, to set the A button in RetroArch for player one, we have to change the config key `input_player1_a_btn`. Snowflake sees this as `ButtonA` instead, and knows to change this key.)

2. Find the equivalent of the device element this property is set to in the native configuration format, from the input mapping files. Continuing with RetroArch as our example, `ButtonA` for an XInput controller is defined as `0` in RetroArch's configuration format.

3. Run it through the serializer and spit out the configuration line `input_player1_a_btn = "0"`.

The input template class also handles setting how the emulator identifies the device because it is also given the exact enumerated input device object in addition to the Layout - Device mappings. In the case of RetroArch, we set `input_device_p1` to the DirectInput enumeration number — the order in which the device was found.

This does require that the emulator adapter plugin supports the device explicitly, more specifically the layout the device's *InputEnumerator* exposes, along with the *InputEnumerator* itself, due to the differences between devices and even OS-level input APIs that report buttons differently. This abstraction allows us to use the same unambiguous mappings for basically every supported emulator. Dolphin uses human readable strings such as `Button A` to represent buttons, as well as paths such as `XInput\0\XInput Controller` to represent IDs. We can get all the information for the device path from the enumerated input device object, which automatically collects this information, and write a mapping for the element `ButtonA` to represent the string `Button A` for serialization.

### Handling the emulated keyboards

As mentioned before, Stone does not support emulated keyboards beyond the semantic `Keyboard` controller element. Should a controller have this element, it is the responsibility of the consumer, in this case, Snowflake, to properly serialize 1:1 keyboard key mappings. As of the time of writing, not even emulators such as PCEm allows one to change the mapping from guest to host keyboard, nor is it encouraged if such a thing were possible. If required, such an emulator adapter should not expose this setting to allow remapping of keyboard layouts, and should not expose any Input Options. 

## Why?

Why keep all this data about layouts and such when there are simpler ways of achieving this, for example, a 1:1 string replace?

As I've mentioned before, Snowflake's input handling has gone through multiple revisions, and this is the one I've settled upon after working on reflection-based configuration generation. It's a bit confusing to wrap your head around multiple abstractions of input, and frankly, it takes myself a few days every time I come back to it after a break to re-acquaint myself with it. However, assigning semantic meaning to both the device and the emulated controller allows the frontend UI to not only infer defaults and make configuration quick and painless but also to display metadata and information about each button. 

For example, the frontend UI could restrict face button mapping to only the face buttons, and allow emulated d-pads to be mapped to directional buttons on a controller. We can also support numeric pads and buttons that aren't part of any standard controller, such as the WonderSwan's flip buttons. Knowing the layout of a user's device opens up so many possibilities for smoother UI integration with emulators. Stone is not the first to come up with this concept, many of the ideas in this document were inspired by how RetroArch handles input across multiple systems. I feel that RetroArch's approach is ill suited for displaying things in a more emulator agnostic way, such that Stone and this input pipeline was written to be even more abstract and high-level than RetroPad. 


## The takeaway — TL;DR

* Snowflake uses the [Stone specification](http://github.com/SnowflakePowered/stone), a database of platform and controller information compiled for general use, not just Snowflake.
* Using abstractions on top of the physical gamepad device, and a defined layout for an emulated controller, we can map a guest controller/gamepad to a host device in a mostly declarative manner during configuration generation.
* The system is flexible enough to work for most if not all emulators, and across a wide range of controllers, physical or emulated.
* Devices must have a plugin that exposes metadata from the OS, and the device layout to Snowflake.
* Frontend UI can take advantage of the layout format to display more information about both the device layout, and the emulated layout.
* Supports multiple profiles across multiple combinations of emulated to a physical gamepad. 
* Drawing the diagram of all this working together is hard...
