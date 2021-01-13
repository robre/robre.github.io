---
layout: post
title: "IOKit Reversing Tips"
categories: hacking reverse-engineering ios iokit
---

## IOKit Reversing Tips
I've been reversing some IOKit drivers for iOS for a couple months now, and I have collected a bunch of tips that would have accelerated my progress if I had known them earlier, so I decided to share them with anyone who wants to get into iOS security research.

### IOreg
My first tip when it comes to IOKit, is to get familiar with ioreg. It's a handy tool to list the IOKit registry, and see what drivers are currently loaded. Use it with the right flags - I like to use it to find information about a specific loaded driver by looking up its name like this: `ioreg -f -r -t -n name`, or by looking up a specific class like this: `ioreg -f -r -t -c IOClass`. The combination of `-r -t`  will print the full tree for your classes clients, and its providers. For example, to check out my Macs display, I can use `ioreg -f -r -t -n display0` to get:
```
+-o Root  <class IORegistryEntry, id 0x100000100, retain 24>
  +-o MacBookAir9,1  <class IOPlatformExpertDevice, id 0x100000114, registered, matched, active, busy 0 (13676 ms), retain 67>
    +-o AppleACPIPlatformExpert  <class AppleACPIPlatformExpert, id 0x100000115, registered, matched, active, busy 0 (8818 ms), retain 52>
      +-o PCI0@0  <class IOACPIPlatformDevice, id 0x10000015d, registered, matched, active, busy 0 (4972 ms), retain 55>
        +-o AppleACPIPCI  <class AppleACPIPCI, id 0x1000001e8, registered, matched, active, busy 0 (4957 ms), retain 39>
          +-o IGPU@2  <class IOPCIDevice, id 0x1000001b3, registered, matched, active, busy 0 (1912 ms), retain 34>
            +-o AppleIntelFramebuffer@0  <class AppleIntelFramebuffer, id 0x10000046d, registered, matched, active, busy 0 (119 ms), retain 21>
              +-o display0  <class IODisplayConnect, id 0x1000005e4, registered, matched, active, busy 0 (1 ms), retain 6>
                | {
                | }
                |
                +-o AppleBacklightDisplay  <class AppleBacklightDisplay, id 0x1000005e5, registered, matched, active, busy 0 (0 ms), retain 9>
```

On MacOS, I recommend getting the IORegistryExplorer.app from [Apples additional Developer tools](https://developer.apple.com/downloads), which provides the same functionality as ioreg, but in a more comfortable visual UI.

### Get IDA Pro
I started reversing with Ghidra, simply because it's free. After some time however I made the switch to IDA Pro, and the difference for iOS is immense. They did a really good job of integrating a lot of iOS specific functionality to IDA in newer versions, such as natively working with the kernelcache, or dyld_shared_cache. I am aware that it is expensive, and I was lucky to be able to use it through my university. 

### Get a symbolicated kernel cache
Normally, Apple removes symbols from the kernel cache, making reversing a bit more tedious. However sometimes symbols are not removed, e.g. in a research kernel. One of those can be found in [this ipsw](https://updates.cdn-apple.com/2020SummerSeed/fullrestores/001-32635/423F68EA-D37F-11EA-BB8E-D1AE39EBB63D/iPhone11,8,iPhone12,1_14.0_18A5342e_Restore.ipsw) (Shoutout to [tihmstar](https://twitter.com/tihmstar) for posting about this apparently secret technique! I didn't know about it before..)

### Accelerate IDA Analysis
The following tweet is self-explanatory:

<blockquote class="twitter-tweet"><p lang="en" dir="ltr">protip: closing all windows within IDA makes decompilation way quicker<br>protip^2: `/Applications/IDA\ <a href="https://t.co/vZWwuTPqWs">https://t.co/vZWwuTPqWs</a> -B kernelcache` -- even quicker</p>&mdash; sparkey (@iBSparkes) <a href="https://twitter.com/iBSparkes/status/1303995748380536833?ref_src=twsrc%5Etfw">September 10, 2020</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

### IDA Kernelcache Scripts
[Brandon Azad](https://twitter.com/_bazad) released an amazing IDA Toolkit for iOS kernelcache analysis. However his version only worked for IDA 6 and up to iOS 12. Fortunately, there is a public fork by cellebrite-srl, porting it to IDA Pro 7.5 and iOS 14: [https://github.com/cellebrite-srl/ida_kernelcache ](https://github.com/cellebrite-srl/ida_kernelcache) 

This script will use the class metainformation found in any IOKit class to create class structures with vtables, making analysis much more comfortable. An example from bazads [blogpost](https://bazad.github.io/2018/03/ida-kernelcache-class-reconstruction/):
Before running ida_kernelcache:
```C
__int64 __fastcall AppleKeyStoreUserClient::registerNotificationPort(__int64 a1, ipc_port *a2, __int64 a3, int a4)
{
    __int64 v4; // x0@2

    if ( *(_BYTE *)(a1 + 248) & 0x10 )
    {
        v4 = *(_QWORD *)(a1 + 216);
        if ( a4 == 43 )
        {
            *(_QWORD *)(v4 + 208) = a2;
            if ( *(_BYTE *)(v4 + 0xE0) )
                sub_FFFFFFF0069D0AF4(v4, 0, 0);
        }
        else
        {
            *(_QWORD *)(v4 + 0xC8) = a2;
        }
    }
    return 0LL;
}
```

After running ida_kernelcache:
```C
IOReturn __fastcall AppleKeyStoreUserClient::registerNotificationPort(AppleKeyStoreUserClient *this, ipc_port *port, unsigned int type, unsigned int refcon)
{
    AppleKeyStore *provider; // x0@2

    if ( this->AppleKeyStoreUserClient.entitlements_flags & 0x10 )
    {
        provider = this->AppleKeyStoreUserClient.provider;
        if ( refcon == 43 )
        {
            provider->AppleKeyStore.system_keybag_update_port = port;
            if ( provider->AppleKeyStore.field_e0 )
                AppleSEPKeyStore::tickle_system_keybag_update_port(provider, 0, 0);
        }
        else
        {
            provider->AppleKeyStore.notification_port = port;
        }
    }
    return 0;
}
```


### Siguzas IOKit tools
Next, I would like to mention [Siguzas](https://twitter.com/s1guza) open source tools, that he released on github:

- [https://github.com/Siguza/iometa ](https://github.com/Siguza/iometa)
- [https://github.com/Siguza/iokit-utils ](https://github.com/Siguza/iokit-utils)

They are some handy tools, mostly to get quick information on classes

### OS X and iOS Kernel Programming Book
[This](https://github.com/sun6boys/Books/blob/master/Os%20X%20And%20iOS%20Kernel%20Programming.pdf) book helped me a lot with understanding the iOS kernel. There are also some other good resources, especially the [official documentation](https://developer.apple.com/documentation/kernel/iokit_fundamentals)

### Check the sources
You probably already know that XNU is open source, but did you know that some IOKit Families, and tools like [ioreg](https://opensource.apple.com/source/IOKitTools/IOKitTools-114.40.1/ioreg.tproj/) are also [open source?](https://opensource.apple.com/release/macos-1101.html)
Some Families even include open source sample driver implementations!

### Know the tools
On MacOS, you will find some additional tools to interact with drivers. I don't find them incredibly useful for reverse engineering, but it's good to know about them:

- kextstat: display loaded kexts
- ioclasscount: count instances of IO Classes
- ioalloccount: summarize IOKit memory usage
- kextfind: find kernel extensions
- kextlibs: find OSBundleLibraries needed by a kext
- kextutil: load and debug kexts

--- 

Thank you for reading, if you know of any tip that I missed, I would love to hear about it!

[@r0bre](https://twitter.com/r0bre), 13.01.2021


