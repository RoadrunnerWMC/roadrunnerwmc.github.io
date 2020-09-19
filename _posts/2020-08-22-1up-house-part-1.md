---
layout: post
title:  "Secrets of the 1UP House"
date:   2020-08-22
categories: blog
---

In this 2-part series, I'll be presenting what is probably the most in-depth analysis of the *New Super Mario Bros.* Green Toad House minigame on the internet.

Part 1 (this one) will describe how the game works, and part 2 will investigate strategies for playing it. While researching for this post, I accidentally fell into the rabbit hole of analyzing NSMB's RNG function, and that ended up [becoming its own separate post]({% post_url 2020-05-08-nsmb-rng %}). It ultimately wasn't directly related to this series, but you can consider it a sort of "part 0."

## The game

If you're unfamiliar with the 1UP minigame, give the video below a quick watch (courtesy of my friend [Meatball132](https://www.youtube.com/channel/UCkhuk_OxMOPUD1-HYls-Few)):

<center><iframe width="384" height="288" src="https://www.youtube-nocookie.com/embed/K9dWmDdzgt4" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen style="margin-top:0.5em;margin-bottom:1.5em"></iframe></center>

There are six blocks. Three contain single 1UP mushrooms, one contains three of them, one contains an "x2" card that doubles whatever you've won so far, and one contains a Bowser card that ends the game. The block order is random and there's no "tell," so this is purely a guessing game.

## Implementation basics

The contents of each block are chosen while the room is being loaded, before it even appears on-screen. The Toadsworth actor itself acts as a sort of minigame-coordinator sprite responsible for arranging the cards, which I personally find to be a hilariously pointlessly skeuomorphic design choice.

He randomly places the 3UP card into one of the blocks, randomly places the x2 card into one of the five remaining blocks, and finally randomly places the Bowser card into one of the four remaining blocks[^bowser_card_twice]. The three leftover blocks implicitly receive single 1UPs.

<div style="text-align: center; margin: 2em 0em 2em 0em;">
<img src="/assets/004-1up-house-part-1/flowchart.png" style="width: 90%; display: inline-block;" alt="A flowchart showing the process described here." title="&rarr; player chooses blocks &rarr; player gets mushrooms &rarr; player eventually closes the game and lives the rest of their life" />
</div>

...Blog post over, right? What more could there be to say about something that simple? Well, it turns out that there are three separate factors that make the game less than fully random. I'll describe them in order of least to most severe.

## Complication 1: choosing a block

Placing a card in one of the six blocks naturally involves choosing a random number 1 through 6 -- or rather, 0 through 5, because computers. Here's the (decompiled) line of code Toadsworth uses to accomplish that:

```c
int choice = (6 * (rand_nsmb() & 0x7fff)) >> 15;
```

``rand_nsmb`` can return any of 4294967296 different values, and that number isn't evenly divisible by 6. Since that range can't be divided into 6 equally sized portions, some choices are inevitably going to be slightly more probable than others. With the code above (which also discards 17 of the 32 bits of randomness, which doesn't help either[^discarding_bits]), the probabilities of picking the first or fourth positions are about 0.003% higher than the others.

Now, obviously, a 0.003% bias is completely negligible -- so much that I won't even be factoring it into further calculations at all[^further_calculations]. I still find it kind of interesting theoretically, though.

The other two complications, though, *do* affect gameplay to significant degrees.

## Complication 2: Toadsworth takes pity

<video autoplay loop muted playsinline style="width: 45%; float: left; margin: 0em 2em 1em 1em" alt="A visualization showing how a card that would be unhelpful to choose on the first turn is surreptitiously moved if it is picked then." title="Toadsworth EXPOSED">
    <source src="/assets/004-1up-house-part-1/pity-rule.webm" type="video/webm" />
    <source src="/assets/004-1up-house-part-1/pity-rule.mp4" type="video/mp4" />
    <source src="/assets/004-1up-house-part-1/pity-rule.ogv" type="video/ogg" />
    <i>Your browser doesn't support HTML5 <code>&lt;video&gt;</code> tags. :( To see my animations, please try viewing this page in a browser that does!</i>
</video>

If the first block you hit contains the Bowser card or the x2 card, Toadsworth discreetly switches that card with one of the 1UP cards just before the animation plays. This makes it impossible to reveal either of those cards on the first turn, guaranteeing that you'll always win at least one 1UP, and that the x2 card (if found) will always double *something*.

Unlike the other factors affecting the minigame's randomness, this one is very intentional.

<div style="clear:left" />

## Complication 3: handling card collisions

<video autoplay loop muted playsinline style="width: 55%; float: right; margin: 0em 2em 1em 2em" alt="A visualization supporting the idea that to prevent multiple cards from being placed in the same block, some strategy for redistributing the probabilities is needed." title="*Boing*">
    <source src="/assets/004-1up-house-part-1/intro.webm" type="video/webm" />
    <source src="/assets/004-1up-house-part-1/intro.mp4" type="video/mp4" />
    <source src="/assets/004-1up-house-part-1/intro.ogv" type="video/ogg" />
    <i>Your browser doesn't support HTML5 <code>&lt;video&gt;</code> tags. :( To see my animations, please try viewing this page in a browser that does!</i>
</video>

Something I've neglected to mention up to this point is how Toadsworth avoids putting multiple cards into the same block. After all, if you just pick a random number 0-5 three times, you could get, say, [1, 1, 4], but obviously the 3UP and x2 cards can't *both* be placed in the second block.

Let's quickly look at some possible correct solutions to this, before seeing Nintendo's actual solution.

<div style="clear:right" />

### Restricting the range

<video autoplay loop muted playsinline style="width: 55%; float: right; margin: 0em 2em 1em 2em" alt="An animated visualization of how the probabilities would be distributed among the question blocks, according to the &quot;restricting the range&quot; method." title="Or you could move conflicting cards directly to the end of the list, but I think that's harder to implement correctly.">
    <source src="/assets/004-1up-house-part-1/range-restriction.webm" type="video/webm" />
    <source src="/assets/004-1up-house-part-1/range-restriction.mp4" type="video/mp4" />
    <source src="/assets/004-1up-house-part-1/range-restriction.ogv" type="video/ogg" />
    <i>Your browser doesn't support HTML5 <code>&lt;video&gt;</code> tags. :( To see my animations, please try viewing this page in a browser that does!</i>
</video>

One way to solve this would be to realize that once you've picked the 3UP card location, instead of picking a random number 0-5 for the x2 card, you should really pick a random number 0-4, since there are only 5 spots left. Then, if the number you get is greater than or equal to the first, add 1.

And just like that, your number is now in the range [0, 3UP card location - 1] ∪ [3UP card location + 1, 5], with a proper discrete uniform distribution. You would then do the same thing to place the Bowser card, too.

<div style="clear:right" />

### Rejection sampling

<video autoplay loop muted playsinline style="width: 55%; float: right; margin: 0em 2em 1em 2em" alt="An animated visualization of how the probabilities would be distributed among the question blocks, according to the rejection sampling method." title="This one was the hardest to animate. Did it turn out OK?">
    <source src="/assets/004-1up-house-part-1/rejection-sampling.webm" type="video/webm" />
    <source src="/assets/004-1up-house-part-1/rejection-sampling.mp4" type="video/mp4" />
    <source src="/assets/004-1up-house-part-1/rejection-sampling.ogv" type="video/ogg" />
    <i>Your browser doesn't support HTML5 <code>&lt;video&gt;</code> tags. :( To see my animations, please try viewing this page in a browser that does!</i>
</video>

Alternatively, you could use a discrete version of <a href="https://en.wikipedia.org/wiki/Rejection_sampling">rejection sampling</a>. After placing the 3UP card, pick a random number 0-5 for the x2 card. If it's the same as the 3UP card location, try again by picking another random number 0-5. Keep doing that until you get a number that's different. And then for the Bowser card, keep picking random numbers until you get one that matches neither of the previous two cards.

This is a little less efficient than the previous solution, since it might take more RNG calls to place the x2 and Bowser cards, but it could be simpler to program. You still get a discrete uniform distribution in the end[^infinite_rng_loop], so it's a valid choice.

<div style="clear:right" />

### Nintendo's solution

<video autoplay loop muted playsinline style="width: 55%; float: right; margin: 0em 2em 1em 2em" alt="An animated visualization of how the probabilities are distributed, according to the method actually used in the game." title="Look at the bright side, though &mdash; Nintendo's dumb code directly led to the creation of this blog post, only fourteen years later!">
    <source src="/assets/004-1up-house-part-1/nintendo.webm" type="video/webm" />
    <source src="/assets/004-1up-house-part-1/nintendo.mp4" type="video/mp4" />
    <source src="/assets/004-1up-house-part-1/nintendo.ogv" type="video/ogg" />
    <i>Your browser doesn't support HTML5 <code>&lt;video&gt;</code> tags. :( To see my animations, please try viewing this page in a browser that does!</i>
</video>

What actually happens could perhaps be described as "rejection sampling done wrong."

After the 3UP card is placed, a random number 0-5 is chosen for the x2 card. If it matches the 3UP card location... the game adds one (wrapping around from 5 to 0 if needed).

This is **bad**; it makes the position immediately to the right of the 3UP twice as likely to be chosen as any of the other spots!

The Bowser card is placed in the same way: if the random number matches either of the first two, it's incremented until it's unique. If the x2 and 3UP are right next to each other (which has a 33% chance of happening), the spot to the right of them is *three times as likely* as the others (50%!) to get the Bowser card!

The pity-rule code also uses the same process to move the x2 or Bowser card, if the player hits one of those blocks first. In this case, all three of the initial card positions are considered ineligible choices.

## Strategizing

Knowing how the game works now, it seems that it might be possible to come up with strategies that could, on average, improve your results. After all, some configurations are more likely to happen than others. Figuring out exactly *how* to do that, though, is somewhat tricky.

Stay tuned for part 2, in which I present the provably optimal strategy for this minigame!

<br />

---
---

<span></span>

[^bowser_card_twice]: To be more precise, he places the Bowser card, and then immediately forgets he did so and does it a second time. This repetition accomplishes absolutely nothing. Nintendo's programming is great.
[^discarding_bits]: It *is* considered good practice to discard the least-significant few bits when using a linear congruential generator ([*"The low-order bits of LCGs when m is a power of 2 should never be relied on for any degree of randomness whatsoever." -- Wikipedia*](https://en.wikipedia.org/wiki/Linear_congruential_generator#Advantages_and_disadvantages)), but Nintendo's actually doing the opposite here by discarding the seventeen *most* significant bits.
[^further_calculations]: In the next blog post, I mean. This one doesn't have very many calculations at all.
[^infinite_rng_loop]: Unless, of course, your RNG function has a bug where it might just repeat the same number over and over again. Then this strategy could make the game freeze. But no game would be silly enough to use a broken RNG function like that! Right? Right??
