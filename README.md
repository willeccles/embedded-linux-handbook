<!-- vim: set spell spelllang=en_us: -->

# Embedded Linux Handbook

This manual or handbook or guide or reference or whatever you want to call it is
intended to simplify the process of building an embedded Linux system.

## What it is

A simple reference for many of the more obscure concepts in embedded Linux,
particularly for beginners. It will tell you the basics of using device trees,
as well as more advanced features of device trees, how device drivers work, and
how to use some of them.

## What it is not

This is not intended to be a tutorial. If you are looking for a tutorial, there
are likely plenty of places to start all over the internet, ranging from
Raspberry Pi forums and guides to BeagleBone and other similar platforms'
documentation and examples. If you're looking for long, detailed, and in-depth
guides on how to use your device's driver from user-space, you are looking in
the wrong place. This is purely intended to be a reference. Everything else can
be found in the kernel's documentation or elsewhere on the web.

## Background

After venturing into our company's first foray in embedded Linux on custom
hardware (specifically using an Octavo Systems OSD3358 SiP), I found that there
was little to no documentation, and as time went on, we were accruing a lot of
technical debt by doing things the wrong way and then hacking to work around it.
This guide is intended to help others avoid this problem, and is based on my
experiences on that project and potentially future ones.

## Topics covered

This guide will talk a little bit about U-Boot, Linux configuration, device
trees, and user-space code using device drivers. It may also touch on the usage
of Linux with RT PREEMPT patches.

# Table of Contents

1. [Basic Concepts of Embedded Linux](basics.md)

