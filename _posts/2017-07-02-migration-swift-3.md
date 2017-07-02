---
layout: post
title:  "The Grand Migration from Swift 2.2 to Swift 3"
date:   2017-07-02 10:03:00 -0700
categories: iOS, Swift
---
We haven't got the chance last year to do this administration-side work due to rapid development, and has been suffering from a major problem: Swift 2 iOS apps can only run on Xcode 7 which is not officially compatible with macOS Sierra or above (the most disturbing thing being the Instruments profiling tool will always crash at launch). Considering Swift 4 is launching officially this fall, it's perfect time now to upgrade our 2.7k+ lines of code first to Swift 3. But making changes to *everywhere* in the entire codebase sounds daunting; you probably don't realize that one character modification to somewhere is breaking other seemingly unrelated components far apart. Also, it's hard to provide a good time estimate on the migration, since Xcode may prompt more syntax errors after you fix some. After getting through the entire process in about 5 days (2 days trial-and-errors and 3 days doing real work), here are the steps I followed and some experiences that might be helpful.

## Test CocoaPods frameworks in a separate project
**The most important principle in software engineering, or any engineering discipline in general, is modularity.** You will sure to be confused if the entire codebase's syntax is converted at once, and some wired compilation errors come up. So let's first start with the CocoaPods frameworks that are used in the project. What I am doing here is to first go to each pod's GitHub page checking its documentation, to make sure the developer already adds in Swift 3 support. If not, be sure to also check the pull requests as other developers may have done the same as they used this framework before. Update `Podfile` of the main project to use the newest version of these frameworks, or select appropriate branches with Swift 3 support.

Then I create a new iOS Swift project in Xcode, add a few simple Swift codes (to make sure it runs in Swift 3) and tests these frameworks *one by one* to make sure they can be built successfully. `git` now comes in handy as we can use a separate branch to test each framework which makes future management easier. If everything go well, the last step would be go back to `master` branch, add all these frameworks in `Podfile` and try building the project, which should succeed if all individual tests are passed. Now we have an exactly same dependency hierarchy as our main project.

But usually things are not flowing so smoothly, so here comes the main part: *What should I do if there are compilation errors for one or more frameworks, or they are not even announced Swift 3-compatible?* Basically there're three situations possible. I will use a simplified version of our project's framework hierarchy for illustration as shown in graphs: first is the one before updating pods and second is the one after update; pods in blue are having compilation errors after update.
![Swift3-Migration-1](/assets/swift-3-migration-1.png "swift-3-migration-1")
![Swift3-Migration-2](/assets/swift-3-migration-1.png "swift-3-migration-2")

### Updating syntax
If **this framework does not have any dependency itself**, we are lucky then: __just adapting its syntax to Swift 3__ would do well. The `Graphs` framework in the graph falls in this case, and turns out it's an easy fix after the Migration Assistant finishing auto-conversions. One interesting thing is that the Swift compiler flags out a segfault after basic conversion: 
```
Swift Compiler Error: command failed due to signal: Segmentation fault: 11.
Stack dump:
0.	Program arguments: /Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/bin/swift -frontend -c ...
1.	While emitting IR SIL function @_TFFC6Graphs12PieGraphView4drawFVSC6CGRectT_L_7convertu0__Rxs8Hashable_S_11NumericTyped__S3_rFTqd__3arrGSaqd___1fFqd__qd___GSaqd___ for 'convert' at /Users/.../Graphs/Graphs/PieGraphView.swift:67:9
```

It's likely to be a Swift compiler bug but we have to workaround it get code running don't we? Locate the problem file and try commenting out different functions until pinpoint the exact code. In this case, the compiler is too confused on a nested functions with generics; Move the `convert<S; NumericType>` function out of the `draw` function makes it happy again:
```swift
internal class PieGraphView<T: Hashable, U: NumericType>: UIView {
	override func draw(_ rect: CGRect) {
		func convert<S: NumericType>(_ s: S, arr: [S], f: (S) -> S) -> [S] {
			// codes
		}
	}
}
```

And your future self will thank you for following the standard practice of modifying a framework:
1. Fork the repository, make changes here and push commits to remote.
2. Modify the `Podspec` of this framework in main project's `Podfile`, redirecting source code to the forked repo. For example, one entry in my `Podfile` looks like: `pod 'Graphs', :git => 'https://github.com/maxwyb/Graphs'`
3. Submit these changes as a pull request to the original repository. Modify the Podspec back to original when the developer accepts the pull requests.

Keep in mind that **never make changes to CocoaPod frameworks in the main project**; they will be lost after running `pod install` or `pod update`. One disadvantage of the above approach is that theoretically, the framework cannot be updated to newer versions before the pull request is accepted. But if this's the case, the pod is probably no longer actively maintained and there may not be any new versions released anyways :-)

### Updating syntax and adapting to different interface
This can happen when **the target framework is dependent on one or more other frameworks**, similar to `SimpleAuth` in the graphs: frameworks dependent on it and frameworks it depend on are all updated to newer versions, but itself is "squeezed in the middle". You might doubt if this causes any further trouble: each framework are keeping their interfaces unchanged during update; the failed framework would be the only concern as others are just "black boxes" doing their own jobs as expected. Sadly this is only the ideal situation in college computer science classes and things get more complicated in the industry: __frameworks' interfaces changes occasionally in major updates, so the code logic of our failed framework need to be modified__. For example, the `ReactiveCocoa` framework, one dependency of `SimpleAuth` and written in Objective-C, is divided into two frameworks `ReactiveSwift` (the mainstream version written in Swift) and `ReactiveObjC` (leaving some legacy Objective-C codes) in months. Adapting to the new interfaces of codes written in another programming language definitely needs more work, as well as some changes to previous bridging header configurations. In the worst case, the legacy `ReactiveObjC` might be added in as well to keep the target framework's integrity.

As you've already seen, this is ugly as the principle of modularity is violated here. Making these semantic changes requires developers to understand what's going on in other frameworks interacting with the target one. This is essentially "writing another framework", so the workload is exponentially increased. Although it's still possible to get it compile and work, I would not recommend pursuing the modification unless absolutely necessary. We are going with the next option in our case: choosing an alternative approach and using no third-party framework for same functionality.

### Using an alternative framework, or switching to another approach in app
Moral of the story: **try only using third-party frameworks with high popularity in large or long-term projects!** It will save you lots of trouble in the fast developing front-end world if these frameworks are actively maintained by the developer.

## Migrate the main project
So far the tough works are all done: compatibility issues of third-party codes are resolved; the only thing left to upgrade are our own codebase which we understand well. It's recommended to stop developing during the entire process of migration (if this's possible; googling "migrating to swift 3 blog" leads to some good articles on how large-scale companies deal with the simultaneous developing, such as [this one](https://engblog.nextdoor.com/migrating-to-swift-3-7add0ce0655)). Also, there may be things going wrong when modifying a large project as a whole, especially when manually re-organizing linked frameworks and libraries. So remember to __commit the whole directory after every operation__, which makes every change revertable.

Install and update the pods, run the Swift Migration Assistant, accept code changes and there will be hundreds of compilation errors popping up. The way we approach them is to first __do some corrections from the beginning to get a feel, and then start from the lines whose changes you completely understand__, including the syntax and usage of new methods. We realize soon that these manual modifications are not as versatile as expected. Each one can be grouped into one of the categories such as optional values unwrapping, variable downcasting, etc.; modifications in one category are essentially the same. Therefore, __modifying codes in the same category together__ can also be a good way to start. Also, be assured that Swift compiler is not as whimsical as we thought before. Based on my experience, more errors would be shown only when a function signature is changed, as the compiler is not able to check inside a function if its declaration has syntax error. These manual code changes are done in less than one day for our codebase.

My general feeling towards Swift 3 is that **optional values unwrapping and type casting are becoming more rigorous**. Below are some typical manual changes in the migration; [Ray Wenderlich's article](https://www.raywenderlich.com/135655/whats-new-swift-3) has a brief overview on these changes.

An example of optional values unwrapping:
```swift
var label: String!
label = "Hello Swift 3!"

// In Swift 3, anotherLabel is of type String? but label is of type String!
let anotherLabel = label  

// quick and dirty way: force unwrapping
switch (anotherLabel!) {
case "Hello Swift 3!":
	print("case 1")
	break;
default:
	break;
}

// program defensively: optional chaining
if let anotherLabel = anotherLabel {
	print(anotherLabel)  // "Hello Swift 3!"
}

// anther syntax if you're pretty sure the value exists
guard (anotherLabel != nil) else {
	print ("unexpected value")
	return  // from current function
}
print(anotherLabel!)  // "Hello Swift 3!"
```

Other examples of strict downcasting and *the Grand Renaming* of function names:
```swift
// Swift 2 followed by Swift 3
let image = imageCache.objectForKey(userName) as? UIImage
let image = imageCache.object(forKey: userName as AnyObject) as? UIImage

transferManager.download(downloadRequest).continueWithExecutor(AWSExecutor.mainThreadExecutor(), withBlock: { (resultTask) -> AnyObject? in
	// codes
}
transferManager.download(downloadRequest!).continueWith(executor: AWSExecutor.mainThread(), block: { (resultTask) -> AnyObject? in
	// codes
}
```

Besides, there are some minor changes of functions and properties in frameworks such as `AWSCore`. For example, the result `AWSTask` from a completion handler no longer has an `exception` attribute. Also, when calling functions with multiple parameters with default values, the order of passed-in arguments is enforced.

## Resolving linker errors and runtime crashes
Up to now our app can be successfully compiled and we are almost good to go. Just one more moment, it may not run on an iOS simulator or real device yet; Xcode may give an error such as `linker command failed with exit code 1 (use -v to see invocation)`. This error is usually caused by our previous adjustments of frameworks used in the project, so linker cannot find some of them now. Go to Project - Build Settings - __Framework Search Paths__ and clean up unused frameworks.

The app can run now, but it may crash before fully launched due to uncaught exceptions. In this case, double check if **manually linked frameworks and libraries are at the newest version**. For example, in our case, an exception is generated by `AWS Mobile Hub Helper` framework in AppDelegate: `[AWSTask exception] unrecognized selector sent to instance 0x........` This is because the Mobile Hub Helper framework is not managed by CocoaPods and the Swift 2 version was manually added in before. Follow Amazon's documentation to download the newest SDKs, add them to **Linked Frameworks and Libraries** (any libraries used in the project, including those come with the operating system) and **Embedded Frameworks** (third-party libraries that have to be embedded in the project) accordingly. Also, properly set up `Info.plist` for authentication and custom shell scripts for building project if necessary. *Be sure not to add duplicates if the framework is already managed by CocoaPods.*

So you may be wondering why Amazon manages all of its iOS SDKs on CocoaPods but not Mobile Hub Helper? Me too :-)

## Test the running app
Okay, everything are up and running now, but not all functions are guaranteed to be working correctly yet. This is the time to "play around with" your "new app" and discover anything that looks wrong. In one section of our app, the UICollectionViews inside a UITableView are not displaying anything. It turns out that `collectionView(_ collectionView: UICollectionView, cellForItemAt indexPath: IndexPath) -> UICollectionViewCell` is never called after a quick debugging. Our first thought was to check if the custom `UICollectionViewDataSource` and `UICollectionViewDelegate` are set up correctly, that is, if we have adapted to the changes of these protocol's new function signatures in Swift 3. This is highly possible to be the problem months ago as old versions of Migration Assistant would fail to upgrade these delegate functions if they're placed in class extensions; however it was not the problem in this case. Also, another delegate function `collectionView(_ collectionView: UICollectionView, numberOfItemsInSection section: Int) -> Int` is always correctly invoked returning non-zero values. The problem turns out to be fairly obscure: the Storyboard contraints of these UICollectionViews and UICollectionViewCells are somehow treated differently in Xcode 8, so not a single cell would be visible on the user interface. Deleting these contraints work as a temporary solution, but the root cause will be explored in the future. 

Ideally, the testing should be easy and deterministic if good __unit tests__ are available, and these are the stuff we are working on. Anyways, the app is now running on the newest (and best) development platform, with the entire set of development tools readily available!

## Reflections
Planning ahead and "thinking before doing" are keys to keep we organized in engineering work, especially when dealing with a large and long-term project. 

Ready for the Migration to Swift 4 this fall?
