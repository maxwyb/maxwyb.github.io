---
layout: post
title:  "Elegance of Swift"
date:   2016-07-08 10:19:00 +0800
categories: iOS
---
After reading through Apple's iOS Developer Library and exploring some iOS apps, I begin to feel the elegance of Swift 2.0, and why many developers refer to it as "the most beautiful programming language yet". Its high-levelness enables some tedious work in C++/Java to be easily done, which often brings a little surprise to me. I am keeping this post to summarize all little features I noticed in Swift that make it exciting.  

- Member function `indexOf()` for arrays  

```swift
var buttonsList = [UIButton]()

func buttonTapped(button: UIButton) {
	print("The tapped button is of index \(buttonsList.indexOf(button)).")
}
```
To check if an object is an element of an array and return its index in C++/Java, I would have to write a loop to traverse the array and do comparisons every iteration. The problem would be even a little more complex if the object is a class/struct consisting of multiple variables. However one line of code does the job in Swift.  

One issue related is that `button` may not necessarily be in the array `buttonsList`; the mechanism of *optional values* in Swift wonderfully solves this problem: a `nil` is returned to the destination if that is the case.