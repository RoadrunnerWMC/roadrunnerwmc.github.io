---
layout: post
title:  "Managing Screen Brightness in Lavalit Tower"
date:   2019-11-06
categories: blog
---

Today I'll be demonstrating a small technique we used in [Newer DS](https://newerteam.com/ds/) to make the brightness effect in Lavalit Tower seem fairly smooth in spite of the Nintendo DS's limitations. I'll avoid going into too much actual technical detail.

**Please note:** I'll be using a modified copy of Newer DS to demonstrate one approach we *didn't* use; videos from this unofficial copy will be marked with "(DEMO)" in the bottom-right corner.


## Screen brightness on the DS

The Nintendo DS provides a register (basically like a variable) known as ``MASTER_BRIGHT`` ("master brightness") that lets games change the screen brightness. A value of 0 produces the usual brightness, -16 is pure black, and 16 is pure white. Since that's not very many values to choose from, the player can definitely tell the difference between them! This prevents games from making a fade that looks smooth at slow speeds (at least when using this feature to do it).

In Newer DS, we used ``MASTER_BRIGHT`` to darken the screen in certain areas and world maps. When brainstorming ideas for the tower in World 8, we came up with the idea to use it to make a level that gets darker the higher you climb. This ended up being Lavalit Tower.

The small number of available brightness levels made this a bit complicated, though. Values above 0 don't darken at all, and we didn't  want to go all the way down to -16 since the player still has to be able to see something. Thus, we decided to only use 10 brightness levels in each of the level's two main zones.

The next thing to decide, then, was how to choose *when* to switch from one brightness level to the next.


## A first attempt

<video autoplay loop muted playsinline style="width:256px;float:right;padding: 1em 0em 1em 2em">
    <source src="/assets/002-lavalit-tower/naive.webm" type="video/webm" />
    <source src="/assets/002-lavalit-tower/naive.mp4" type="video/mp4" />
    <source src="/assets/002-lavalit-tower/naive.ogv" type="video/ogg" />
    <i>Your browser doesn't support HTML5 <code>&lt;video&gt;</code> tags. :( To see my animations, please try viewing this page in a browser that does!</i>
</video>

The most obvious solution would be to just divide the zone up into 10 sections vertically, and change the brightness level whenever Mario moves from one to another.

On the right, you can see a video of me playing Lavalit Tower on a modified copy of Newer DS that uses this strategy.

I've added some annotations in post to make it clear what's going on. The red line is Mario's Y position, the white lines are the dividers, and the numbers are brightness levels. I've also added some blank space above and below the screen to make the divider lines a little easier to follow.

As you'd expect, whenever Mario crosses a section boundary, the brightness changes. Simple.

Overall, this works fairly decently, but there's a problem...

<div style="clear:right" />

<video autoplay loop muted playsinline style="width:256px;float:left;padding: 1em 2em 1em 0em">
    <source src="/assets/002-lavalit-tower/naive-problem-demo.webm" type="video/webm" />
    <source src="/assets/002-lavalit-tower/naive-problem-demo.mp4" type="video/mp4" />
    <source src="/assets/002-lavalit-tower/naive-problem-demo.ogv" type="video/ogg" />
    <i>Your browser doesn't support HTML5 <code>&lt;video&gt;</code> tags. :( To see my animations, please try viewing this page in a browser that does!</i>
</video>

As shown on the left here, the issue is that the screen will flicker whenever the player repeatedly crosses a transition point. Here, the transition point happens to fall right on the springboard, making for a very unpleasant visual experience when the player bounces on it.

To fix this, you might consider manually adjusting the exact cutoff points so that they're never right above the ground. Even if you did this, though, there'd still be a flicker if Mario were to jump over a transition line in midair and immediately fall back across it again. Manually positioning the dividers just can't fully solve this problem.

<div style="clear:left" />


## The better approach

<video autoplay loop muted playsinline style="width:256px;float:right;padding: 1em 0em 1em 2em">
    <source src="/assets/002-lavalit-tower/actual.webm" type="video/webm" />
    <source src="/assets/002-lavalit-tower/actual.mp4" type="video/mp4" />
    <source src="/assets/002-lavalit-tower/actual.ogv" type="video/ogg" />
    <i>Your browser doesn't support HTML5 <code>&lt;video&gt;</code> tags. :( To see my animations, please try viewing this page in a browser that does!</i>
</video>

Here's what Newer DS actually does.

Conceptually, we use a sort of sliding-window technique to decide when to change the brightness. Whenever Mario goes above the window, the screen gets darker and the window moves upward. When he goes below, it gets lighter and the window goes down.

The crucial feature is that the window is *significantly taller* than the amount by which it moves up or down. This means that if Mario goes above a dividing line and immediately falls back below it, the screen won't go back to its previous brightness right away, because the bottom of the window is still quite a bit lower than the line he just crossed. The same thing happens if Mario dips below a brightness-increasing line and hops back over it.

And that's how we made it so that you can't flicker the screen in Lavalit Tower by hopping in place.
