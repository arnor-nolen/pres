---
title: Wayland scaling
author: Andrei Gorbulin
date: 2023-04-13
styles:
    style: gruvbox-dark
---

# Wayland protocol

Wayland is a communication protocol that specifies the communication between a display server and its clients.

The Wayland protocol is built from several layers of abstraction.
* Wire protocol.
* XML specification.
* Some broader patterns which are frequently used (naming conventions, design decisions, etc.). 

## Wire protocol

A stream of messages decodable with interfaces agreed upon in advance. Usually, Unix sockets. Multiple implementations are possible, however, in practice, `libwayland`'s implemenation is used everywhere.

```
┌──────┐     ┌───────────┐     ┌──────────┐
│Client│ <-> │Unix socket│ <-> │Compositor│
└──────┘     └───────────┘     └──────────┘
```

# XML specification

Higher level procedures for enumerating interfaces, creating resources which conform to these interfaces, and exchanging messages about them — the Wayland protocol and its extensions.

XML document describes what kind of messages are possible to exchange between client and compositor.

There are two types of messages that are possible:
* Events - sent by Compositor.
* Requests - sent by Client.

XML specs are usually split into separate interfaces. Each interface have an integer version attached to it (`v1`, `v2` and so on). Interfaces are used to interact with specific functionality that XML spec provides.

## Generating code

Client-side and server-side code can be generated from these XML specifications by using different tools.

Most projects generate `.h` and `.c` files using `wayland-scanner` (comes with `libwayland`), but other implementations are technically possible.

You can write your own, non-official specs, but it's ultimately up to both client and compositor if they will implement it.

# Demo time

Show an example of an XML protocol structure.

# Integer scaling

*NOTE:* All of the scaling information is related to Core Wayland protocols.

## Obtaining preferred scaling from Compositor

You can obtain integer scaling information from `wl_output v2` interface via `scale` event.

> If it is not sent, the client should assume a scale of 1.

> A scale larger than 1 means that the compositor will automatically scale surface buffers by this amount when rendering.

## Setting buffer_scale of your application

You can set your application's scaling information by using `wl_surface v3` interface via `set_buffer_scale` request.

> This request sets an optional scaling factor on how the compositor interprets the contents of the buffer attached to the window.

> A newly created surface has its buffer scale set to 1.

> Note that if the scale is larger than 1, then you have to attach a buffer that is larger (by a factor of scale in each dimension) than the desired surface size.

# Fractional scaling problem

Client and Compositor can only communicate integer scaling values. If the compositor is running at 1.25 scaling (125%), then the application can never know about it since there is no mechanism of communicating this between them.

However, there's a workaround.

If client somehow knows it should be running at 1.25 scaling, it may set the buffer scale to 2 and render itself at 200% into this buffer. The compositor will make sure to scale it down to correct 125%.

This is an enormous waste of resources and the picture might look a bit blurry, but this has been the only way to scale fractionally for a long time.

*NOTE:* This trick has been used by QT and other apps, that were trying to do fractional scaling in Wayland.

# Fractional scaling support

At 29.11.2022 a `wp-fractional-scale-v1` protocol has been announced as a part of `wayland-protocols 1.31`.

This introduces a way for Compositors to ask Client to be rendered at a specific fractional scale.

## Obtaining preferred scaling from Compositor

In order to obtain fractional scaling information from Compositor, you will need to:
* Create a `wl_surface` add-on object by using `wp_fractional_scale_manager v1` interface via `get_fractional_scale` request.

> Create an add-on object for the the wl_surface to let the compositor request fractional scales.

* Obtain fractional scaling by using `wp_fractional_scale v1` interface via `preferred_scale` event.

> Notification of a new preferred scale for this surface that the compositor suggests that the client should use.

> The sent scale is the numerator of a fraction with a denominator of 120.

# Setting buffer_scale of your application

In order to correctly set Client buffer, you will need to:

* Create a `wl_surface` add-on object by using `wp_viewporter v1` interface via `get_viewport` request.

> Instantiate an interface extension for the given wl_surface to crop and scale its content.

* Set correct rendering dimensions by using `wp_viewport v1` interface via `set_destination` event.

> If the destination size is set, it causes the surface size to become dst_width, dst_height. The source (rectangle) is scaled to exactly this size. 

> The buffer size is calculated by multiplying the surface size by the intended scale.

> The wl_surface buffer scale should remain set to 1.

# Fractional scaling adoption

Fractional scaling introduced: 29.11.2022.

Since the protocol is new, the adoption has happened recently.

|Name|Date|
|---|---|
|Sway & wl-roots|13.02.2023|
|KDE Plasma|14.02.2023|
|Mutter|05.03.2023|
|Blender|01.04.2023|
|GTK|05.04.2023|
