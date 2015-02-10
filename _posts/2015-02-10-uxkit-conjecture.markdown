---
layout: post
title: "UXKit Conjecture"
date: 2015-02-10 15:10:00
tags: [UI, iOS, OS X]
---
I know absolutely nothing about [UXKit](http://sixcolors.com/post/2015/02/new-apple-photos-app-contains-uxkit-framework/), but here are a couple of thoughts:

#### Unified UI framework

What many developers seem to be hoping for is that UXKit is a unification of UIKit from iOS and AppKit from OS X. It would certainly make it easier to move code between these two platforms.

This could be done as a compatibility layer that sits on top of both frameworks, or as a new thing that completely breaks away from UIKit and AppKit.

Is cross-platform compatibility alone really worth the effort, though? Here are a few things that may make it into a new UI framework:

#### Multithreaded UI

Both UIKit and AppKit are very dependent on the main thread, and this hurts performance.

However, almost all CPUs have multiple cores these days. Rewriting UIKit and AppKit to be more like [AsyncDisplayKit](http://asyncdisplaykit.org) just seems to make sense. 

AsyncDisplayKit feels like a project that is just waiting to be sherlocked. It would be silly if Apple *wasn't* working on something like that.

UXKit could be a rewrite that takes advantage of multi-core CPUs.

#### Swift

AppKit and UIKit are both written in Objective-C and depend heavily on its dynamic messaging capabilities. Swift does not have such features, but is supposed to be the replacement of Objective-C.

UXKit could be a redesign using a static, type-safe language that has native support for things such as closures and generics.

#### Cruft

AppKit goes back to the stone age and UIKit isn't quite the spring chicken anymore either. In hindsight, some design decisions are questionable -- for example, why is navigation a function of view controllers?

UXKit could be a cleanup of this technical debt.
