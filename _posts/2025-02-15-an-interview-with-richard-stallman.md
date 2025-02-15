---
layout: post
title: An interview with Richard Stallman
date: 2025-02-15 2:00:00 -0800
categories: programming special
---

This post will contain a few excerpts from an interview with RMS conducted by me and my friend last year.

Originally, the interview was for a school project, but I believe it shows some general cool information about GNU, their design choices, and their philosophy that people would (hopefully) like to hear about.

This post won't include the entire 50-minute interview nor it's transcript - if you want it, contact me and I'll be more then happy to provide them.

### Prelude

First, some background, and a disclaimer.

This interview was recorded with the intent of being incorporated into a National History Day project on the Linux Kernel (Torvalds, unfortunately, never responded to our email). Because of this, some questions were about how Linux tied into GNU. 

Getting the interview was somewhat easy - aside from the fact that conventional calling methods would not work. We hopped into the FSF's mumble server to conduct it. Because of a combination of this and our janky audio setup (we didn't have a speaker), there might be some echos. I've cut out some long pauses and tried to get rid of the background noise, but it isn't perfect.

My first impression of RMS as a person was that he was very clear-cut in all his beliefs, and knew his priorities. But also, he recognized the shortcomings of GNU and the FSF and the reality of the world. One cannot deny that he is a powerful man.

A disclaimer is that I do not publicly share political worldviews with Stallman, and I take no ownership of his words. 

Each excerpt will be a short description, followed by some audio clips and transcripts of our questions and his responses.

### Excerpts

If you want to get to the juicy parts about ideology and such, you can skip over the sections about the design philosophies of UNIX and GNU Hurd.

---

#### On the interactions between GNU and the Linux Kernel, as well as some intuition into the fundamental goals of the GNU project.

/assets/stallman/gnu-linux-relation.wav

"The Linux Kernel completed GNU. Looking purely at GNU/Linux, which do you think is more important, GNU or the Linux Kernel?"

> The Linux is a kernel. That's all that Linux is. **Linux was the last missing piece that completed the goal of the GNU project**, which was to make a complete free operating system, the GNU operating system. 

> Now, **Torvalds didn't think of his work as part of the GNU system**. He didn't associate with the GNU project. He didn't even talk to us. 

> **But in fact, it filled the last gap in what we had been working on for most of a decade. And it's because of having a complete free operating system that there was a change in what people, a fundamental change in what people could do and have freedom. Before that, the only operating systems were non-free**. Their developers controlled what the users could do because users couldn't change the operating systems. So the decisions of what the programs would let you do and what they wouldn't let you do were made by the companies that owned the systems...

---

#### On a history of proprietary systems, as well as what motivated RMS to create the FSF.

/assets/stallman/toughest-quote.wav

"You mentioned that, up until GNU, every operating system was proprietary. Am I correct on that?"

> **That's true if you look at, say, the world starting from 1980 or so. Because earlier there were other free operating systems. In 1970, just about every operating system was free. And likewise, in the mid-70s**, initially when there were first personal computers, their operating systems were free also. And the computers sold by Digital Equipment Corporation in the 1960s and early 70s, they came with free operating systems too. 

> **These examples taught me the moral difference between living under a proprietary software regime and living in a free world in your computer.** The difference between having freedom and not was marked for us, for people like us who experienced living in freedom. **And this is what taught me the importance of free software** when basically, when free software slipped away from us, or our freedoms slipped away under the influence of non-free software, this was a tremendous loss. 

> **And I took this so seriously, so much to heart, that I decided I'm going to win back our freedom if it's the last thing I do.**

---

#### On why UNIX was the model for GNU, and some components developed for it.

/assets/stallman/unix-1.wav

"The Unix Operating System was developed in the 1970's, Am I correct?"

> **Yes, and it was non-free, officially non-free, although for some of the, for the late 70s and early 80s, copies of the source were unofficially distributed**. 

> It was never officially approved to do so, but Bell Labs didn't try very hard to suppress it. But later things got stricter with Unix. I'm telling you about things that people have told me.

> I never used Unix back then. **The first time I ever used Unix was around the start of 1984 when I was starting to explore it and see if my decision to make a Unix-like operating system was a good decision. It turned out that Unix wasn't an ideal model, but it was good enough. So I continued with that plan**

/assets/stallman/unix-2.wav

"You said that Unix wasn't the model system - wasn't the best, but it was enough?"

> I wouldn't say, I didn't use the word best. **It wasn't ideal**. It wasn't quite, some of the, some technical aspects of it were not quite as good as I hoped, but they were good enough ... **it was still a useful model to imitate**. Remember, what I was doing, **the plan I had made, was to imitate Unix by developing free replacements for the 200 or so components of Unix**.

/assets/stallman/unix-3.wav

"So what components would you say were the most essential that had to be compatible with Unix?"

> **The C library, in some ways the compiler, and a whole bunch of little utilities that scripts would use**. Having scripts is not something, scripts is something that not every operating system had. **That was one of the particularly good aspects of Unix.**

> **That it had a powerful scripting language.** Of course, the implementation of that language had to be upward compatible. We made a shell that was upward compatible with the Unix shell. **The Unix shell was called the Bourne Shell. so ours was called the Bourne Again Shell.**

"So, it was a combination of these factors that made you chose UNIX over something, like, say MS-DOS."

> **Right. Well, of course, MS-DOS was so pitiful it wasn't even worth thinking about. Unix was a multi-user time-sharing system, and it's clear that that's what we wanted.** There were other multi-user time-sharing systems that I could have chosen to imitate, but Unix was clearly better then the other possible choices.

---

#### On GNU Hurd.

/assets/stallman/gnu-hurd.wav

"The team that was working on GNU, the operating system; you've mentioned on your website that they were planning to release GNU Hurd, which they are still working on, if I'm correct."

> Right. **The GNU Herd ran into serious technical difficulties.** The aim was to a collection of, to replace a kernel with a collection of servers that would run on a microkernel. And **we started with a microkernel called Mach that had been developed as a funded project at Carnegie Mellon.** 

> But **we tried to implement the upper layers of the operating system in an advanced modular way where the top half of the kernel would consist of various independent programs running under time sharing on top of the microkernel. That turned out to have serious problems which nobody ever fixed.**

> Some volunteers still work on the new herd, **I've more or less given up hope that it will ever be our main kernel.**

---

#### On the motivation behind GNU, and the divide between free and open source software.

/assets/stallman/motivation-1.wav

"Nowadays, a lot of GNU slash Linux distributions are associated with the open source ideology. Aiming to highlight like the practical advantages of programs. What's your sentiment towards how GNU slash Linux and free or open source software evolved and will evolve in the near future?"

> **Well, in the early 90s, there was a philosophical disagreement between parts of the free software community. There were the people who supported the free software idea, which was a moral idea.** Users deserve to have control of the programs they run, of programs that do their computing, because it's an injustice if they don't. 

> So, **we didn't develop the GNU system just because we thought it would be nice to develop a system and have it become widely used and be admired. We developed it so we and you could have freedom.**

> **Now, this was too radically shocking for some people, and they were more conservative in their ways of thinking, and they didn't want to rock the boat.** They didn't want to... to make such a moral criticism of the usual ways of doing things. They, you know, some people who held those views did contribute to developing free software for other motives. But they were not supporters of the free software movement.

/assets/stallman/motivation-2.wav

"So right now ... there's slightly more open source then free software in the world"

> I think so. **The definition of open source was derived from the definition of free software, but it doesn't look very similar.** Here's what happened. There were these two different philosophies, and they existed within the free software community. 

> **At the beginning of 1998, the people who disagreed with us invented a term they could use to refer to their philosophy and not even hint at the existence of ours.** they could promote their philosophy without suggesting to people that there was any other. And **they got the backing of companies with money that could do advertising.**

> As a result, ever since then, it has been an effort for us to make people aware that the Free Software Movement exists. Well, **Free Software is a movement. It is a campaign to win certain freedoms. It's a campaign in the domain of political philosophy, just as a democracy as a way of life is the goal of a movement.** 

> There are many different such political movements with different goals. However, they coined the term open source to describe the practices at a practical level only and not raise any questions of right and wrong, not raise any political questions at all. So they simply, the **open source supporters took for granted the political views normal in our society and didn't raise the idea of rejecting any of them.**

---

#### On the future of the free software movement **NOTE: the audio cuts out in the middle of his response for some time, so the clip is cut**

/assets/stallman/future.wav

"So, what direction do you think that the free software movement, as you just described it, what direction do you think that will be going in in the near future?"

> **I can't see the future. And I don't try because I know that generally I can't. Lots of unpredictable things have happened during these four decades. Some have been very disappointing, but that's the nature of life.** Things happen by surprise. I would never have guessed many of the things that have happened around me. So I couldn't answer that. 

> **And besides, I think it's the wrong kind of question to ask. Our society teaches people to be obsessed with the question of what's going to happen and overlook the question of what should happen.** 

> You'll find lots of discussion on television about the question of who's going to ... and other media about who's going to win the next presidential election. And not as much serious discussion of who should win ... So **I'm more interested in what we should do than what's going to happen.**

--- 

#### On the uses of GNU/Linux

/assets/stallman/chocolate.wav

"So one last question ... GNU slash Linux, it's really popular. I think that's an undisputed fact. So what would you say are some of its major impacts in the modern world, both from a philosophical sense and a practical sense?"

> From a practical sense, I it's mostly being used in data centers and websites. **And I'm happy that it's good for that, but I really wanted it to become popular on people's personal computers so that individual humans would have freedom.** Unfortunately, it's not so popular for that.

> Just a second.

> Actually, it's inconsiderate and inconvenient of me to **eat some of this chocolate** right now. I should stop until we're done. But it's hard to resist.

> Anyway, so... so **I think still its most important impact is in the idea of software freedom, the idea that the users deserve to control the programs that do their computing. Most people, of course, don't support this and have never even heard of it, but there are people who do, who have, and people who continue to hear about these ideas.**