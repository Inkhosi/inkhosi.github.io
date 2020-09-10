# Linux Shared Object Hijacking

As the title suggests this post covers ways to go about hijacking shared object execution on Linux. Although this is a widely covered topic I thought I would make start here as not everyone is familar with it on Linux.

So what am I talking about? Well, Linux has it's own version of DLLs called Shared Objects which are part of the ELF( Executable and Linkable Format) binaries that are what run natively in Linux. They contain code that is designed to be shared and used in other binaries. There are two ways that shared objects can be used. The first is loaded whilst the binary is being loaded. The second is dynamically loaded during runtime. We will explore what we can do here.

## Objectives

1. Gain code execution through the use of shared object hijacking that doesn't require sudo rights.
2. Hide our tracks by continuing smooth operation

## Example Library

I have provied an example library if you wish to try this [ExampleSharedObject](https://github.com/Inkhosi/ExampleSharedObject). The library simply provides a way to prove you have achieve the exploit by writing a file to your home directory.

## On the hunt

TODO
