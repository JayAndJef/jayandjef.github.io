---
layout: post
title: "The need for quality code"
date: 2025-02-13 8:20:00 -0800
categories: programming
---

I've noticed that some younger developers don't write what most people would consider "clean" code. And it's not only due to inexperience as well.

This post was inspired by one of my latest endeavors, which was to migrate a web application initially developed by some students for my own needs.

I'll go over a few reasons for unclean code, and a way to help fix it.

### Why is some code unclean?

Inexperience is the most obvious, yet not exactly the most common case. Imagine someone who has only worked with single-file python apps before, suddenly needs to write a flask backend for a school website. Although this is a great exercise, the code probably isn't going to come out separated into loosely-coupled modules, with documentation written for each individual handler. Most of the time, however, this isn't the case.

The actual reason why unclean code exists is because **the developers didn't care enough for it** when programming. We didn't try enough to make it more accessible to incoming maintainers, or maybe we were too focused on **making something that works, fast** that we never stopped and thought about it. A lot of times, we know that there's a better way to do something: a better way to make the compiler happy, a better way to structure the component tree, a better way to separate the responsibilities of that module, a better way to document the code for later use. 

But we don't do that. Implementing anything takes time, and we want to save as much of that as possible, right?

### Trying to make it better

I need to first make sure I'm not trying to be a hypocrite - we all have, at some point, committed pieces of code that aren't the best it could be. That one function implementation that could have been faster, or that one algorithm which ignores the edge case of the key not existing in the map, or that one block that `.unwrap()`s way too many times. And we probably all will keep making these mistakes.

But, the problem with having a mindset of write now, fix later is that we aren't preparing our code for the future. In a few weeks, that function *will* catch an edge case, causing your app to panic. In a few months, that library we are dependant on releases a subtle change, causing a multitude of errors to propagate throughout out codebase as we try to remember what the heck this block of code actually did while trying to fix it.

The solution to this is to always try to **look towards the future**. Every time we implement something, we need to ask ourselves if we have truly made it easier for our future selves and future contributors, because the reality is that as the codebase grows, the components also do. If not handled correctly, this leads to more and more complexity - more and more points of failure.

This is also exactly why companies and projects sometimes have strict rules code must comply - they try to rule out classes of errors (the most obvious one being developer misunderstanding), minimize the ones that can't be erased, and make them easier to fix when they pop up.

If anyone reads this, please tell me if there is something you disagree on, or any improvements I could make.



