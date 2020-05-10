---
layout: post
title:  "Default font for foreign characters in Google Chrome"
date:   2020-05-09 6:30:00 -0700
categories: Chrome web CSS Windows
---

Interestingly, Google Chrome on Windows 10 renders foreign characters in a different font compared to Mozilla Firefox and Microsoft Edge. "Foreign characters" here refer to characters that are not from the language declared in the HTML header.

```html
<!-- English -->
<html lang="en">
<!-- Simplied Chinese -->
<html lang="zh-cn">
```

The default font for each language can be changed inside Chrome's settings using the [Advanced Font Settings](https://chrome.google.com/webstore/detail/advanced-font-settings/caclkomlalccbpcdllchkeecicepbmbm) extension. However, fonts of these characters in our case are not because the HTML is declared to be in a different language from the characters. 

Below is an example of some Simplied Chinese characters in an English HTML document. Apparently Google Chrome uses a serif font (Ex. SimSun) but Firefox and Edge use a sans-serif font (Ex. Microsoft YaHei). Chrome on macOS has the same behavior as FireFox and Edge on Windows. In my opinion, a sans-serif font is more readable especially on lower-resolution screens.

![browsers-fonts](/assets/chrome-firefox-edge-font.jpg)

Seems like the default font in Windows 10 for Simplied Chinese is Microsoft YaHei too, so Chrome may have its own internals to manage these defaults.
