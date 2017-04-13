---
layout: post
title:  "Symbolicate a Crash Log from the App Store Review"
date:   2017-02-20 22:26:00 -0700
categories: iOS
---
Yesterday we managed to symbolicate some crash logs of an iOS app from the App Store submission review; briefly summarize the process and some issues here for future reference.

### Preparation 
The following files are needed for symbolication:
1. The *exact same* `.app` file that you submitted to the App Store, which can be found in Xcode's archive Organizer. (StackOverflow guys said so, but apparently the executable inside is not accessed by the `symbolicatecrash` script)
2. The `.dSYM` debug symbol files corresponding to this submitted archive. These debug symbols should be downloaded from Apple because Apple may recompile your app to optimize performance when bitcode is enabled in the project settings. The download is also done in the archive Organizer.
3. **The symbol files from a real device with the same kind as the testing device (iPad or iPhone), running the exact same iOS version. (Eg. iOS 10.3.1)** Plug in the device and open you project in Xcode. Remember the long waiting of "Processing symbol files" prompt you might have got before? Xcode copies these device-specific symbol files onto your computer: `~/Library/Developer/Xcode/iOS DeviceSupport`. *Therefore, you have to get a real device similar to the testing environment used by the Apple review team, in order to do the symbolication.* They are necessary to run the `symbolicatecrash` script, but are rarely mentioned online.
4. The crash logs, whose filenames must end with `.crash`.

### Process
1. Open the Device manager in Xcode and drag the crash logs onto the testing device connected. Xcode will symbolicate the crash log for you by running the following script underneath.
2. Manually run the `symbolicatecrash` Perl script inside the Xcode application: `/Application/Xcode-7.3.1/Contents/SharedFrameworks/DVTFoundation.framework/Versions/A/Resources/`. The script could be passed in with arguments specifying file locations, but I doubt these options are parsed by the script at all. Apparently it just locates each necessary symbol files by doing a Spotlight search (Ahh...), so it's a good idea to *copy the above files to a location that will be indexed by Spotlight*.

```bash
./symbolicatecrash -v --dsym MyApp.app.dSYM crashlog.txt MyApp-1.04-Build15.app/MyApp > symbolicate-crash.txt 2> symbolicate-crash-stderr.txt

./symbolicatecrash -v crashlog.txt MyApp.app.dSYM > symbolicate-crash.txt 2> symbolicate-crash-stderr.txt
```

### A Note on UUIDs
By comparing the UUID in the crash log with that of the app executable and debug symbol, we make sure they are corresponding to each other: different archives built with even the same source code would have different UUIDs. *Technically* symbolication can only be performed when the UUIDs of these three things are the same.  

We can check the UUIDs by `dwarfdump --uuid` command on macOS. *DWARF* (debugging with Attributed Record formats) is a debugging file format used by many compilers and debuggers to support source-level debugging, according to [IBM](https://www.ibm.com/developerworks/aix/library/au-dwarf-debug-format/). See the Apple's webpage attached below.  

Interestingly, **the binary image's UUID in Apple's crash logs does not match with the UUID of our local archive executable and dSYM**, but we are sure the executable used here is the one submitted to the App Store. The UUID of the `.app` executable and that of the dSYM debug symbols match together, as seen below. However, somehow the script still managed to symbolicate the crash logs, and the result looks correct. Yet this might be the reason that we are getting only partially symbolicated logs as discussed below.

```bash
$ dwarfdump --uuid MyApp-1.04-Build15.app/MyApp
UUID: 9413156C-52BA-3338-855A-D8E15AF8689B (armv7) MyApp-1.04-Build15.app/MyApp
UUID: FCEB5855-F695-305B-B779-C3203E8E5B25 (arm64) MyApp-1.04-Build15.app/MyApp

$ dwarfdump --uuid MyApp.app.dSYM/
UUID: 9413156C-52BA-3338-855A-D8E15AF8689B (armv7) MyApp.app.dSYM/Contents/Resources/DWARF/MyApp
UUID: FCEB5855-F695-305B-B779-C3203E8E5B25 (arm64) MyApp.app.dSYM/Contents/Resources/DWARF/MyApp

$ grep --after-context=2 "Binary Images:" crashlog.txt
Binary Images:
0x1000e8000 - 0x100607fff MyApp arm64  <050e7575a3433274ba64cd4e856a3d3f> /var/containers/Bundle/Application/BD79B5D2-934E-4339-9D8A-E3691FBE5562/MyApp.app/MyApp
...
```

### Why My Crash Log is only Partially Symbolicated?
For example, if you get something like this:
```
Thread 0 name:  Dispatch queue: com.apple.main-thread
Thread 0 Crashed:
0   libsystem_kernel.dylib        	0x000000018cd53260 0x18cd52000 + 4704
1   libdispatch.dylib             	0x000000018cc415e8 0x18cc2d000 + 83432
2   libdispatch.dylib             	0x000000018cc40ca0 0x18cc2d000 + 81056
3   StoreKit                      	0x000000019ae3aafc 0x19ae37000 + 15100
4   myApp                       	0x0000000100189670 PaymentsDisplay.requestProductInfo() -> () (PaymentsDisplay.swift:97)
5   myApp                       	0x00000001001893b4 PaymentsDisplay.viewDidLoad() -> () (PaymentsDisplay.swift:31)
...
```
Apparently all function calls to the system libraries are not symbolicated. Unfortunately this is the current situaiton of us right now. Someone mentioned that it is caused by **the script not finding correct device symbols**, which may be the case because we are unable to get the exact kind of testing device used by Apple (who knows if they use an iPad Air 2 or iPad Pro or even iPad 2?). But this is probably already good enough for debugging your own code in the project.

### Resources
[Understanding and Analyzing Application Crash Reports --Apple](https://developer.apple.com/library/content/technotes/tn2151/_index.html)  
[Symbolicating iOS Crash Reports --StackOverflow](http://stackoverflow.com/questions/1460892/symbolicating-iphone-app-crash-reports)  
[How to Match Crash Report to a Build? -- Apple](https://developer.apple.com/library/content/qa/qa1765/_index.html)

