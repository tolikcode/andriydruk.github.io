+++
date = "2015-09-05T14:14:19+03:00"
description = ""
draft = false
image = "/img/material-003.png"
tags = ["androiddev"]
title = "Bonjour in Android applications"

+++

**What is Bonjour?** 
Wiki determination of Bonjour is

> Bonjour is Apple's implementation of Zero-configuration networking (Zeroconf), a group of technologies that includes service discovery, address assignment, and hostname resolution. Bonjour locates devices such as printers, other computers, and the services that those devices offer on a local network using multicast Domain Name System (mDNS) service records.

Apple determination is

> Bonjour, also known as zero-configuration networking, enables automatic discovery of devices and services on a local network using industry standard IP protocols. Bonjour makes it easy to discover, publish, and resolve network services with a sophisticated, yet easy-to-use, programming interface that is accessible from Cocoa, Ruby, Python, and other languages.

In another words, is the software component that used for discovering another devices (PC, Mac, smartphones, printers, etc) in network via all available interfaces. In terminologies of Bonjour this called "service". Any application in operation system can register service and assign it to opened port on this computer (actually Bonjour not guarantee that port in service's metadata is opened and connected to app that registered this service). All services are registered in some domain (that's mandatory parameter of all services and you can find domain naming conventions [here](https://developer.apple.com/library/mac/documentation/Cocoa/Conceptual/NetServices/Articles/domainnames.html)). Also applications can discover all available services in network(s) at any domain. You should understand that Bonjour is technology only for discovering services, all connection you should do by yourself anyway.

You can find more information about Bonjour in [official documentation](https://developer.apple.com/library/mac/documentation/Cocoa/Conceptual/NetServices/Introduction.html).

<!--more-->

**What about Android? Can we use this powerful technology in ours Android projects?** Yes. Google use this technology with a lots of their projects. For example: you try to connect to Chromecast in your living room: it's Bonjour. You are playing games with friends via wifi: it's Bonjour. You print documents via wifi on your printer: it's Bonjour.

**There is standard Android API for it?** Yes. Google provides [Network Service Discovery API](http://developer.android.com/training/connect-devices-wirelessly/nsd.html) that use the same technology that Apple uses in Mac OS X and iOS (Oh, I forget to tell you that Bonjour is open-source). This API are  available from API level 16 (Android 4.1) and works in general well, but it not covered all Bonjour API. One of the most important part of Bonjour API is ability to share some metadata about service, also known as TXT records. 

That what [official documentation](https://developer.apple.com/library/mac/documentation/Cocoa/Conceptual/NetServices/Articles/NetServicesArchitecture.html) says about it:

> The TXT record has the same name as the corresponding SRV record, and can contain a small amount of additional information about the service instance, typically no more than 100–200 bytes at most. This record may also be empty. For example, a network game could advertise the name of the map being used in a multiplayer game, and a chat program could advertise the availability of the user (for example, idle, away, or available). If you need to transmit larger amounts of data, the host should establish a connection with the client and send the data directly.

It can be very useful and I can't understand why this feature are not available in official SDK, because there are no technical reason for it.

**If there are any other options?** Yes, another option for Android developers is [jmDNS](http://jmdns.sourceforge.net) library. It's open-source library and  is programmed in Java 1.6 (because for Android developers it matter) and is compatible with Apple's Bonjour. You can use it at any Android API level and this library supports TXT records. But I have some issue with jmDNS (I got few crashes inside jmDNS from Crashlytcics, and I have no ideas how I can fix it) and it work pretty slow.

Also Google (not Android open source project) introduced a new cross-platform library [Google Nearby API](https://developers.google.com/nearby/). This API consists of two conceptually different API: Nearby Messages API and the Nearby Connections API. Message API can be used for discovering services via all available interfaces (Wifi, Wifi-direct, Bluetooth 2.0 and LE) and connect two or more devices via Internet. Actually it's very interesting idea in time of fast mobile internet (Maybe I will write about it more in another article). And Connections API can be used for discovering and connection via wifi interface. Actually it's really good solution. You shouldn't care about sockets, ports, discovering, connections and multithreading. All this features have been already done. But this solution work only with Android and iOS, and it's not open-source (that's why I pay attention about Google above). If you want discover some services from another system or your app works with another libraries (for example: printers or vnc servers) you should look to another options.

**Do you have any idea?** Yes, and it why I write this article. I have good news for you: you can just get Apple's implementation of Bonjour and use it in your Android project. 

**WHAAAAAAT? Apple's libraries in Android project?** Yes, it called mDnsResponder and it have been programmed at C many years ago (actually Apple bought it). Look at architecture overview from [documentation](http://opensource.apple.com/source/mDNSResponder/mDNSResponder-66.3/README.txt?txt): 

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

Actually Android uses the same solution with one system daemon and lots of applications. This daemon is available for developers since API 16 (Android 4.1) and you can connect to it from native level via dns-sd.h.

**But I don't know C/C++! do you have any Java wrapper for it?** No, but Apple has. Let's compile it together. First of all you should compile mDnsResponder native client and you can find source code [here](https://android.googlesource.com/platform/external/mdnsresponder/). It's part of android open-source project and updated from time to time, but it is completely independent library that work through POSIX API. I believe, that you can use any version of it with any version of Android. Also you should add JNI bridge to this native source code. Just go to dns-sd.h file and look for version of mDNSResponer, in my case it's:

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

It means that it's mDNSResponder 320.10.80. Now go to [Apple Open Source](http://opensource.apple.com/tarballs/mDNSResponder/) site and find this version. Unzip archive and found folder /mDNSShared/Java/. Copy JNISupport.c file to you jni folder and all *.java files to your scr folder. Then add JNISupport to Android.mk file, in my case:

~~~makefile
LOCAL_SRC_FILES :=  mDNSShared/dnssd_clientlib.c  \
                    mDNSShared/dnssd_clientstub.c \
                    mDNSShared/dnssd_ipc.c \
                    mDNSShared/JNISupport.c
~~~

That's all: compile you native code via command 'ndk-build' and compile Android project. 

And one more thing: Android can stop daemon for save battery or for any other reasons if it doesn't used. That's why you should call next method before working with any code from com.apple.dnssd.*.

~~~java
context.getSystemService(Context.NSD_SERVICE);
~~~

Otherwise, mDNSResponder will throw checked exception "DNSD-SD Daemon not available".

**Do you have any examples?** Yep, you can find example of using mDNSResponder (it contains compiled native libraries with last ndk version for all architectures) in my project Bonjour Browser, all code available on [GitHub](https://github.com/andriydruk/BonjourBrowser). Also you can look on it in action on [Google Play](https://play.google.com/store/apps/details?id=com.druk.bonjour.browser&hl=uk) and it works really fast.