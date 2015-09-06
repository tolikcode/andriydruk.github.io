+++
date = "2015-09-05T14:14:19+03:00"
description = ""
draft = false
image = "/img/material-003.png"
tags = ["androiddev"]
title = "Bonjour in Android applications"

+++

**What is Bonjour?** 
Wiki definition of Bonjour is

> Bonjour is Apple's implementation of Zero-configuration networking (Zeroconf), a group of technologies that includes service discovery, address assignment, and hostname resolution. Bonjour locates devices such as printers, other computers, and the services that those devices offer on a local network using multicast Domain Name System (mDNS) service records.

Apple definition is

> Bonjour, also known as zero-configuration networking, enables automatic discovery of devices and services on a local network using industry standard IP protocols. Bonjour makes it easy to discover, publish, and resolve network services with a sophisticated, yet easy-to-use, programming interface that is accessible from Cocoa, Ruby, Python, and other languages.

In other words, Bonjour is a software component that is used for other devices discovery (PC, Mac, smartphones, printers, etc) in a network via all available interfaces. A Bonjour term for a device on a network is "service". Any application in your operating system can register a service and assign it to an opened port on your computer (actually Bonjour does not guarantee that a port in service's metadata is opened and connected to the app that registered this service). All services are registered in some domain (that's a mandatory parameter for all services and you can find domain naming conventions [here](https://developer.apple.com/library/mac/documentation/Cocoa/Conceptual/NetServices/Articles/domainnames.html)). Also applications can discover all available services at any domain. You should understand that Bonjour is only a technology for services discovery, all connection you have to do by yourself anyway.

You can find more information about Bonjour in [official documentation](https://developer.apple.com/library/mac/documentation/Cocoa/Conceptual/NetServices/Introduction.html).

<!--more-->

**What about Android? Can we use this powerful technology in our Android projects?** Yes. Google uses this technology in lots of their projects. For example: you try to connect to Chromecast in your living room - it's Bonjour. You are playing games with your friends via wifi - it's Bonjour. You print documents via wifi on your printer - Bonjour is here again.

**Is there a standard Android API for it?** Yes. Google provides [Network Service Discovery API](http://developer.android.com/training/connect-devices-wirelessly/nsd.html) that uses the same technology Apple uses in Mac OS X and iOS (Oh, I've forgotten to tell you that Bonjour is open-source). This API is available starting from API level 16 (Android 4.1) and works well in general, though it doesn't cover all Bonjour functionality. One of the most important parts of Bonjour API is ability to share some metadata about a service, also known as TXT records. 

That's what [official documentation](https://developer.apple.com/library/mac/documentation/Cocoa/Conceptual/NetServices/Articles/NetServicesArchitecture.html) says about it:

> The TXT record has the same name as the corresponding SRV record, and can contain a small amount of additional information about the service instance, typically no more than 100–200 bytes at most. This record may also be empty. For example, a network game could advertise the name of the map being used in a multiplayer game, and a chat program could advertise the availability of the user (for example, idle, away, or available). If you need to transmit larger amounts of data, the host should establish a connection with the client and send the data directly.

This feature can be very useful and I don't understand why it isn't available in the official SDK. There is no technical reason for this.

**Are there any other options?** Yes, another option for Android developers is [jmDNS](http://jmdns.sourceforge.net) library. It's an open-source library written in Java 1.6 (this matters for Android developers) and it's compatible with Apple's Bonjour. You can use it at any Android API level and this library supports TXT records. Yet I do have some issues with jmDNS (I got a few crashes inside jmDNS from Crashlytcics, and I have no idea how I can fix it) and it works pretty slow.

Also Google (not Android open source project) has introduced a new cross-platform library [Google Nearby API](https://developers.google.com/nearby/). Its API consists of two conceptually different APIs: Nearby Messages API and Nearby Connections API. Message API can be used for services discovering via all available interfaces (Wifi, Wifi-direct, Bluetooth 2.0 and LE) and connecting two or more devices via Internet. Actually this is a very interesting idea in time of fast mobile internet (I might write more about it in another article). And Connections API can be used for discovering and connecting via wifi interface. Actually it's a really good solution. You don't need to bother with sockets, ports, discovering, connections and multithreading. All these have already been done for you. But this solution works only with Android and iOS, and it's not open-source (that's why I emphasize that it's a Google project). If you want to discover some services from another system or your app works with other libraries (for example: printers or vnc servers) you should look for other options.

**Do you know any other options?** Yes, and that's why I'm writing this article. I have good news for you: you can just get Apple's implementation of Bonjour and use it in your Android project. 

**WHAAAAAAT? Apple's libraries in Android project?** Yes, it's called mDnsResponder and it has been written in C many years ago (actually Apple has bought it). Here is its architecture overview from [documentation](http://opensource.apple.com/source/mDNSResponder/mDNSResponder-66.3/README.txt?txt): 

> A typical mDNS program contains three components:

![Alt text](/img/Screen Shot 2015-09-05 at 23.21.23.png)

> The "mDNS Core" layer is absolutely identical for all applications and
all Operating Systems.

> The "Platform Support" layer provides the necessary supporting routines
that are specific to each platform -- what routine do you call to send
a UDP packet, what routine do you call to join multicast group, etc.

> The "Application" layer does whatever that particular application wants
to do. It calls routines provided by the "mDNS Core" layer to perform
the functions it needs --
 * advertise services,
 * browse for named instances of a particular type of service
 * resolve a named instance to a specific IP address and port number,
 * etc.
The "mDNS Core" layer in turn calls through to the "Platform Support"
layer to send and receive the multicast UDP packets to do the actual work.

> Apple currently provides "Platform Support" layers for Mac OS 9, Mac OS X,
Microsoft Windows, VxWorks, and for POSIX platforms like Linux, Solaris,
FreeBSD, etc.

> Note: Developers writing applications for OS X do not need to incorporate
this code into their applications, since OS X provides a system service to
handle this for them. If every application developer were to link-in the
mDNSResponder code into their application, then we would end up with a
situation like the picture below:

![Alt text](/img/Screen Shot 2015-09-05 at 23.21.31.png)

> This would not be very efficient. Each separate application would be sending
their own separate multicast UDP packets and maintaining their own list of
answers. Because of this, OS X provides a common system service which client
software should access through the "/usr/include/dns_sd.h" APIs.

> The situation on OS X looks more like the picture below:

![Alt text](/img/Screen Shot 2015-09-05 at 23.21.39.png)

> Applications on OS X make calls to the single mDNSResponder daemon
which implements the mDNS and DNS-SD protocols.

In fact Android uses the same solution in one system daemon and lots of applications. The daemon is available for developers since API 16 (Android 4.1) and you can connect to it from a native level via dns-sd.h.

**But I don't know C/C++! Do you have any Java wrapper for it?** No, but Apple does. Let's compile it together. First of all you need to compile mDnsResponder native client. You can find its source code [here](https://android.googlesource.com/platform/external/mdnsresponder/). It's a part of Android open-source project and it's updated from time to time, but it is completely independent library that works through POSIX API. I believe, that you can use any version of it with any version of Android. Also you should add JNI bridge to this native source code. Just go to dns-sd.h file and look for version of mDNSResponer. In my case it is:

~~~c
/* _DNS_SD_H contains the mDNSResponder version number for this header file, formatted as follows:
 *   Major part of the build number * 10000 +
 *   minor part of the build number *   100
 * For example, Mac OS X 10.4.9 has mDNSResponder-108.4, which would be represented as
 * version 1080400. This allows C code to do simple greater-than and less-than comparisons:
 * e.g. an application that requires the DNSServiceGetProperty() call (new in mDNSResponder-126) can check:
 *
 *   #if _DNS_SD_H+0 >= 1260000
 *   ... some C code that calls DNSServiceGetProperty() ...
 *   #endif
 *
 * The version defined in this header file symbol allows for compile-time
 * checking, so that C code building with earlier versions of the header file
 * can avoid compile errors trying to use functions that aren't even defined
 * in those earlier versions. Similar checks may also be performed at run-time:
 *  => weak linking -- to avoid link failures if run with an earlier
 *     version of the library that's missing some desired symbol, or
 *  => DNSServiceGetProperty(DaemonVersion) -- to verify whether the running daemon
 *     ("system service" on Windows) meets some required minimum functionality level.
 */

#ifndef _DNS_SD_H
#define _DNS_SD_H 3201080
~~~

It means that mDNSResponder version is 320.10.80. Now go to [Apple Open Source](http://opensource.apple.com/tarballs/mDNSResponder/) site and find this version. Unzip downloaded archive and find folder /mDNSShared/Java/. Copy JNISupport.c file to your jni folder and all *.java files to your scr folder. Then add JNISupport to Android.mk file, in my case:

~~~makefile
LOCAL_SRC_FILES :=  mDNSShared/dnssd_clientlib.c  \
                    mDNSShared/dnssd_clientstub.c \
                    mDNSShared/dnssd_ipc.c \
                    mDNSShared/JNISupport.c
~~~

That's all. Compile the native code with 'ndk-build' command and then build your Android project. 

And one more thing: Android can stop the daemon to save battery or for some other reason(if it's not used for example). That's why you should call next method before working with any code from com.apple.dnssd.*.

~~~java
context.getSystemService(Context.NSD_SERVICE);
~~~

Otherwise, mDNSResponder will throw a checked exception "DNSD-SD Daemon not available".

**Do you have any examples?** Yep, you can find an example of mDNSResponder usage (containing compiled native libraries with  the latest ndk version for all architectures) in my project Bonjour Browser, all code is available on [GitHub](https://github.com/andriydruk/BonjourBrowser). You can also see it in action in [Google Play.](https://play.google.com/store/apps/details?id=com.druk.bonjour.browser&hl=uk) It works really fast!