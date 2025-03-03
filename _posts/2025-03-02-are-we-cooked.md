---
layout: post
title: Are we cooked because of AI?
date: 2025-03-02 18:20:00 -0800
categories: development
---

> I can laugh at all my CS major friends because they're all gonna be replaced by AI soon

This is a quote from a year ago by my friend. At that time, the pressure of what seemed like everyone fearing AI was pretty intense. Sure, a lot of experienced people say that something like that would never happen, but I never had the experience to actually test their claim.

Now, a year later, I can finally give some of my (somewhat random) thoughts about this topic.

Simply put, we aren't cooked. 

AI just isn't smart enough to handle the complexity of any sort of project that isn't a trivial website or python script just yet. And, unless something extraordinary happens, it probably won't in the near future as well. Most of the people that say "But chatgpt was able to do *X* for me!" say it because that *X* falls into either 
- a simple web application with thousands of examples online already
- or single-file script or algorithm.

The truth is that LLMs right now are just fancy text generators that are able to compose sentences from their training data and spit it out. If the problem has been encountered before many times, or something very similar has, then they are able to help. Otherwise, they become a waste of time; I've once spent (big mistake) almost an hour trying to debug a build error with gemini until I read the docs and was able to fix it in 5 minutes. 

LLMs are great at some things though, and one can use them to do menial tasks - copilot agent helped me change the style of my entire frontend and normalize spacing and margins. They can generate context-aware boilerplate extremely fast, and can write documentation for functions which are easy to parse.

They can also quickly summarize information on a given topic, and leave more time for thinking to the developer.

But talk is cheap. Show me the code!

When I started on what, to rust maintainers, would be a somewhat trivial [issue](https://github.com/rust-lang/rust/issues/132802) of switching the distribution pipeline for `wasm` targets to use `clang` instead of `gcc`, I had no clue how a dockerfile worked, nor the difference between all the wasm targets. The related parts of the dockerfile looked somewhat like this:

```dockerfile
# make the container, install packages

ENV \
    AR_x86_64_unknown_fuchsia=x86_64-unknown-fuchsia-ar \
    CC_x86_64_unknown_fuchsia=x86_64-unknown-fuchsia-clang \
    CFLAGS_x86_64_unknown_fuchsia="--target=x86_64-unknown-fuchsia --sysroot=/usr/local/core-linux-amd64-fuchsia-sdk/arch/x64/sysroot -I/usr/local/core-linux-amd64-fuchsia-sdk/pkg/fdio/include" \
    CXX_x86_64_unknown_fuchsia=x86_64-unknown-fuchsia-clang++ \
    # more targets...
    CC=gcc-9 \
    CXX=g++-9

# blah blah blah

RUN curl -L https://github.com/WebAssembly/wasi-sdk/releases/download/wasi-sdk-25/wasi-sdk-25.0-x86_64-linux.tar.gz | \
  tar -xz
ENV WASI_SDK_PATH=/tmp/wasi-sdk-25.0-x86_64-linux

# blah blah blah

ENV TARGETS=$TARGETS,aarch64-unknown-fuchsia
ENV TARGETS=$TARGETS,wasm32-unknown-unknown
ENV TARGETS=$TARGETS,wasm32-wasip1
ENV TARGETS=$TARGETS,wasm32-wasip1-threads
ENV TARGETS=$TARGETS,wasm32-wasip2
ENV TARGETS=$TARGETS,wasm32v1-none
ENV TARGETS=$TARGETS,sparcv9-sun-solaris
```

First, I learned the basic dockerfile syntax by asking copilot to explain what each line did. It wasn't able to explain what each environment variable did in the larger scope of the ci system, however, so I needed to take a look at the runner script for the containers. It was able to tell me, however, where the built targets should appear after I ran the container locally. After that, and verifying that the issue exists, I went on to trying to figure out how to change the dockerfile to use `clang`.

Somehow, I could not get the issue to reproduce on any other wasm target besides `wasm32-unknown-unknown` and `wasm32v1-none`. What confused me more, looking at the dockerfile, was that this strange `WASI_SDK_PATH` environment variable was set. A quick google search pointed me at the docs for wasm-wasip1 which said that it used the sdk to build for targets, and the AI summary told me that it used clang internally.

Now, it just took one look at the target names to realize that the issue only persisted on the two bare `wasm` targets because those didn't use the wasi sdk, and were therefore falling back to the default `CC` variable, which was set to gcc. 

Setting the specific compiler environment variables for those two targets fixed the issue. Copilot, when prompted with the correct context spanning all related files, could not come up with this simple solution.

Maybe I'm just repeating what this reddit user had to say on the topic.

> The first rule of being a software engineer is to be adaptable. Computers and technology are always changing, usually at a breakneck pace. If you can't adapt and learn to utilize new technology, you are not suited to this field. No, AI isn't gonna replace us. Anyone that thinks that is delusional and has no understanding how technology works. AI will just be another tool in our toolbelt. What you can do is start experimenting with it now. Get CoPilot and start learning how you can integrate it with your workflow in order to make your job easier. Don't fear change, instead always be looking for how you can better exploit that change for your own benefit.


