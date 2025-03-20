---
title: 'Automating Bug Bounties: The new plan'
date: 2025-03-21 07:00:00 +13:00
---

If you're an avid follower of this brand new blog I've just created (how?!), you will of course know that I have been working on an automated tool for fuzzing web applications. I've explained some more my approach in this [previous blog post](https://sam.roque-worcel.com/benteveo/2022/02/21/automating-bug-bounties.html).

Fuzzing, for those who don't know, is the art of throwing garbage at an application with the intention of determining whether it has a vulnerability. Some would argue that it is a science, but I personally think it's hard to throw garbage in a scientific manner. Some other people may wonder how throwing garbage into an application may lead to the discovery of software vulnerabilities, but that is a whole 'nother topic.

The big idea here is that if we can automate fuzzing, we can create a machine that takes CPU and other computing resources in and produces vulnerabilities as an output, which can hopefully be translated into bug bounty rewards. Lots of people are trying to do this at the moment, as it can give the illusion of being a sustainable way of being your own boss, particularly for people who benefit greatly from an income in american dollars.

I have done bug bounties, also under the illusion of self-employment under the capitalist hellscape that is the world today, and have quite a bit to say about it. So this will be a series of blog posts discussing the design, implementation, horrible mistakes, refactoring and outcomes of creating the above described tool.

### Previous iteration

I started working on the previous iteration of this tool, linked above, several years back during the heights of covid. At that time, similarly to most people on earth, I was slightly insane. In contrast now, also similarly to most, I've completely lost my mind. 

Because of this, and also because of other new perspectives that I've gained after working as a platform engineer for three years in the meantime, there are several changes that I am contemplating. 

To give a high-level overview of each component, in case it is useful to readers, these are the components of my planned solution:

* 