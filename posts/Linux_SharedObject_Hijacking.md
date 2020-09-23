# Linux Shared Object Hijacking

As the title suggests this post covers ways to go about shared object hijacking on Linux. Although DLL hijacking is a widely covered topic, I thought I would make a start here as not everyone is familar with it on Linux.

So what am I talking about? Well, Linux has its own version of DLLs called Shared Objects which are part of the ELF( Executable and Linkable Format) binaries that run natively on Linux. They are libraries that contain code that is designed to be shared and used in other binaries. There are two ways that shared objects can be used. The first is loaded whilst the binary is being loaded. The second is dynamically loaded during runtime. We will explore what we can do here. Before we get to that, I want to explain why you would want to do this. The main reason I see for this is persistance on the Linux machine once you have attained access. It simply gives you a way to have your code execute (at the same permission level) when the vulnerable application is run.

## Objectives

1. Gain code execution through the use of shared object hijacking that doesn't require sudo rights.
2. Explanation on how to hide our tracks by continuing smooth operation

## Operating System and Software

 - [Ubuntu 18.04 LTS](https://releases.ubuntu.com/18.04/)
 - [totem video player 3.26.0](https://wiki.gnome.org/Apps/Videos)

## Example Shared Object Library

I have provided an example library if you wish to try this [ExampleSharedObject](https://github.com/Inkhosi/ExampleSharedObject). The library simply provides a way to prove you have achieved the exploit by writing a file called pwned to your home directory. It makes use of the [GCC common function attribute **constructor**](https://gcc.gnu.org/onlinedocs/gcc-8.4.0/gcc/Common-Function-Attributes.html#Common-Function-Attributes) to achieve code execution on load. The library helps us with objective 1, but can be very noisy as it doesn't care about the host program execution and can cause warnings and errors to show in log files. We will use this for now though.

### Building the library

1. Ensure you have g++ and make installed
2. Clone the repository
3. Open a terminal in the cloned repository
4. Type `make`

The `libexample.so` will be available in the bin directory.

## On the hunt for a vulnerable application

To be able to carry out the shared object hijacking we first need to find an application that is vulnerable to it. So how do we tell if a program is vulnerable? There are different ways to go about this. We can look for shared objects that the program is trying to load and can't find. We can look at the shared objects it uses and check if we have permissions to replace them. Alternatively, look for a directory that allows the loading of plugins. Using the program [strace](https://linux.die.net/man/1/strace) is a straightforward way of finding what shared objects a program is using or looking for when running. It is a program that allows you to trace system calls and signals from a running process. We will use this to look for vulnerable or missing shared objects. As an example the program I will be looking at is the [totem video player](https://wiki.gnome.org/Apps/Videos)

### Using strace

`strace -e trace=openat -o ~/trace.log totem`

Example Output:
```
openat(AT_FDCWD, "/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 3
openat(AT_FDCWD, "/usr/lib/x86_64-linux-gnu/libtotem.so.0", O_RDONLY|O_CLOEXEC) = 3
openat(AT_FDCWD, "/usr/lib/x86_64-linux-gnu/libgobject-2.0.so.0", O_RDONLY|O_CLOEXEC) = 3
openat(AT_FDCWD, "/usr/lib/x86_64-linux-gnu/libglib-2.0.so.0", O_RDONLY|O_CLOEXEC) = 3
openat(AT_FDCWD, "/usr/lib/x86_64-linux-gnu/libgio-2.0.so.0", O_RDONLY|O_CLOEXEC) = 3
openat(AT_FDCWD, "/usr/lib/x86_64-linux-gnu/libX11.so.6", O_RDONLY|O_CLOEXEC) = 3
openat(AT_FDCWD, "/usr/lib/x86_64-linux-gnu/libgtk-3.so.0", O_RDONLY|O_CLOEXEC) = 3
openat(AT_FDCWD, "/lib/x86_64-linux-gnu/libc.so.6", O_RDONLY|O_CLOEXEC) = 3
openat(AT_FDCWD, "/usr/lib/x86_64-linux-gnu/libgstpbutils-1.0.so.0", O_RDONLY|O_CLOEXEC) = 3
openat(AT_FDCWD, "/usr/lib/x86_64-linux-gnu/libgstreamer-1.0.so.0", O_RDONLY|O_CLOEXEC) = 3
... trimmed output ...
```

Looking through the full output for the program, all the shared objects require root-level access to edit. Even where we find a shared object that is missing (`libgail.so`), all the places where it is looked for require root-level access.

```
...
openat(AT_FDCWD, "/lib/x86_64-linux-gnu/libgail.so", O_RDONLY|O_CLOEXEC) = -1 ENOENT (No such file or directory)
...
openat(AT_FDCWD, "/usr/lib/x86_64-linux-gnu/libgail.so", O_RDONLY|O_CLOEXEC) = -1 ENOENT (No such file or directory)
...
openat(AT_FDCWD, "/lib/libgail.so", O_RDONLY|O_CLOEXEC) = -1 ENOENT (No such file or directory)
...
openat(AT_FDCWD, "/usr/lib/tls/libgail.so", O_RDONLY|O_CLOEXEC) = -1 ENOENT (No such file or directory)
...
openat(AT_FDCWD, "/usr/lib/libgail.so", O_RDONLY|O_CLOEXEC) = -1 ENOENT (No such file or directory)
...
```
This presents us with a problem as we are looking to exploit the application without requiring root-level access. If we did have root privileges then we could look to exploit this as a means of persistence. For now we will have to look elsewhere. Looking further in the output we see:

```
...
openat(AT_FDCWD, "/home/user/.local/share/totem/plugins", O_RDONLY|O_CLOEXEC) = -1 ENOENT (No such file or directory)
...
```
This is promising as we usually have write access to the user home directory. Looking at the totem documentation we have some good news:

>Videos supports plugins written in C (and any variation of it), Python, and Vala. To install new plugins, unpack your plugins in ~/.local/share/totem/plugins/ 

[https://wiki.gnome.org/Apps/Videos/Documentation](https://wiki.gnome.org/Apps/Videos/Documentation)

Armed with the knowledge that it will load plugins written in C we can look to package our shared object as a totem plugin.

## Wrap up the exploit

To wrap up our shared object as a plugin we need to ensure the plugin has the same format as totem expects. The easiest way to do this is to copy the plugin format of an existing plugin. An example should be available from:

```
/usr/lib/x86_64-linux-gnu/totem/plugins/apple-trailers/apple-trailers.plugin
```

Create an example.plugin file from the existing .plugin file. Ensure that it has Name and Module fields set to example. E.g:

```
[Plugin]
Module=example
IAge=1
Builtin=true
Name=example
```

We can then create the plugins directory with:

`mkdir -p ~/.local/share/totem/plugins`

Next we create the appropriate plugin directory structure and copy in our files, as shown below.

```
user@ubuntu:~/.local/share/totem/plugins$ tree .
.
└── example
    ├── example.plugin
    └── libexample.so

1 directory, 2 files
```

## Running the exploit

To have the exploit run simply run totem(sometimes called Videos in the GUI).

Here we see the pwned file.

```
user@ubuntu:~$ tree -L 1
.
├── Desktop
├── Documents
├── Downloads
├── examples.desktop
├── Music
├── Pictures
├── Public
├── pwned
├── snap
├── Templates
├── trace.log
└── Videos
```

Contents of pwned

```
user@ubuntu:~$ cat pwned 
Greetings!
```

Every time the totem video player is opened this will now execute. Here we have achieved objective 1 where we have shared object code execution without the use of root-level access. It hasn't been straightforward thanks to the good programming practices of the totem programmers, we only achieved the exploit by mis-using the design decision to allow plugins.

## Hiding our tracks

Although we achieved execution, we haven't done it silently.

totem warnings:
```
(totem:6859): libpeas-WARNING **: 21:10:51.260: Failed to get 'peas_register_types' for module 'example': 'peas_register_types': /home/user/.local/share/totem/plugins/example/libexample.so: undefined symbol: peas_register_types

(totem:6859): libpeas-WARNING **: 21:10:51.261: Error loading plugin 'example'
```

In order to avoid warnings and errors like this we would need to code up the correct plugin interface in the shared object. If we knew we were looking to exploit the plugins folder from the start through intel on our target. We could look to modify existing opensource plugins, hiding our persistence code among a normal looking plugin. 

With that objective 2 is complete. I leave the implementation as an exercise for the reader :-).

# Protecting against shared object hijacking

What is the reason it was a challenge to achieve the exploit?

The answer is the use of rpath in the linker to specify the search directories for the shared objects:

```
ld -rpath=dir
```
In combination with correct linux file permissions, the access to the directories for the libraries is restricted. Since a regular user could not access the search directories we couldn't exploit the missing `libgail.so` without first gaining root-level access.

If you wanted more in-depth protection you would have to look at making use of [code signing](https://en.wikipedia.org/wiki/Code_signing). This has the distinct disadvantage of making user-plugins expensive to develop and would likely prohibit the average user from attempting it. Ergo in the case of the totem video player the first option makes sense, was implemented correctly and the design decision to execute plugins would have accepted the risk.


Updated: 23/09/2020 Fixed spelling and added clarity. Thank you for the proofreading [Philippe Lasnier](https://www.linkedin.com/in/philippelasnier/)
