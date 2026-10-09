+++
title = "Unstuk"
description = "An offline Android app that fixes your phone from a plain complaint, like \"my phone doesn't ring\". A small model on the phone picks the fix, and ordinary code makes it."
weight = 1

[taxonomies]
tags = ["AI", "On-device ML", "Android", "Kotlin", "Python"]

[extra]
local_image = "projects/unstuk/icon.png"
social_media_card = "projects/unstuk/screens.png"
mermaid = true
+++

Ask someone who isn't into phones what's wrong, and you'll hear "it doesn't ring" or "it's talking to me", not "Do Not Disturb is on" or "TalkBack is enabled". The fix is usually one switch. Finding it means knowing what it's called and where this phone keeps it.

**Unstuk** is an app for that. You type what's wrong in your own words, and it finds the switch and flips it. A small model on the phone only works out which problem it is, from a list of 15. Ordinary code does the rest, and reads the phone again before saying it worked. It runs fully offline, since broken internet is one of the things it fixes.

#### [Get the preview APK](https://github.com/rishikeshvk/unstuk/releases) • [Read the write-up](https://github.com/rishikeshvk/unstuk/blob/main/docs/writeup.md) • [Code](https://github.com/rishikeshvk/unstuk) {.centered-text}

<video class="phone-loop" autoplay loop muted playsinline poster="unstuk-loop-poster.jpg">
  <source src="unstuk-loop.webm" type="video/webm">
  <source src="unstuk-loop.mp4" type="video/mp4">
</video>

## What it does

![Four replies from the app: the ringer fixed and checked, a fix that asks before switching the ringer, a question about which screen problem it is, and a polite decline for booking a cab](screens.png)

Type "my phone doesnt ring anymore" and it checks Do Not Disturb, the ringer and the ring volume, fixes the one that's off, and shows you what it checked. If it's less sure, it asks before changing anything. If the complaint could mean a few things, it asks which one. And if you ask it to book a cab, it says it didn't catch that instead of guessing.

## Only acting when it's sure

The model never writes text that gets run or shown. It picks from the list and says how sure it is, and that decides what happens next:

{% <mermaid> %}
flowchart LR
  A[Your complaint +<br/>the phone's settings] --> B[Model picks<br/>a problem]
  B --> C{How sure?<br/>How risky?}
  C -->|sure, harmless fix| D[Fix it]
  C -->|less sure| E[Ask first]
  C -->|unclear| F[Ask which,<br/>or say no]
  E --> D
  D --> G[Check the phone again]
{% </mermaid> %}

So the confidence has to mean something. A model that's 99% sure and wrong would change a setting nobody asked about. I tuned it so that when it says 90%, it's right about 90% of the time, and it only acts alone above 75%. Riskier fixes always ask first, and anything destructive is never automated.

## Actually flipping the switch

This turned out to be the hard part, so I built it before any model. Android lets an app change very few settings by itself, so each fix tries the easiest route it has: change it directly (brightness, the ringer), open the right Settings screen for you to tap (Wi-Fi), tap it for you through an accessibility service (airplane mode, TalkBack), or walk you through the steps.

Nothing counts as fixed until the app reads the phone again and sees the change. In every test on a real phone so far, it hasn't once said something worked when it didn't. It's only fully tested on my Moto Edge 30, though, and other phone makers label their switches differently.

## Testing it

Before training anything, I set up the test: about 600 complaints written by ChatGPT, Gemini and a Claude that knew nothing about the project, locked away before any training data existed. They have typos, Indian English, long rambling stories, and traps like "my ring finger hurts". I scored simple approaches on it first, like keyword matching, so the model had something to beat.

It did beat them. It gets about 89% of complaints right, against 74% for the best simple approach, and answers in under 10 ms on the phone. Two things didn't go the way I wanted:

- I left three problems out of training on purpose, to see if a new problem could work from its description alone. The untrained encoder handles those better than my model does. Mine turns some of them away as "not something I can help with", just because they're unfamiliar. So a new problem now needs a few dozen examples.
- Most of its mistakes are on Gemini's lines, which say things like "the glass" for the screen. Claude wrote all the training data, and it shows.

The bigger gap is that every test so far is on complaints written by AI. Next is getting real people to type what they'd say, and the model stays only if it still wins there. Until then the APK is a preview.

## What I learned

**Test it on the phone, not just the laptop.** The shrunk-down model matched perfectly on my laptop. On the phone it disagreed on 8 complaints. One cause was how I batched lines while testing, and the other was a difference in how the laptop's chip and the phone's chip do the maths. Only comparing answers line by line on the actual phone caught it.

**The thresholds mattered more than the training.** I tried training the model to be unsure about vague complaints. That didn't help. Moving the line where it decides to ask did.

**Libraries bring things with them.** ONNX Runtime quietly added the internet permission to my offline app, and I only noticed at the first release build. Now the build fails if it ever comes back.

**Decide when to give up before you start.** Every experiment had its bar and its stop rule written down first. A few ideas I liked got dropped because of that, and I'm fine with it.

## How it's built

- **App:** Kotlin and Jetpack Compose, with an accessibility service for the tapping
- **Model:** bge-small, a 33M-parameter text encoder, fine-tuned and shrunk to 35 MB, run with ONNX Runtime
- **Training:** Python, on Colab
- **Catalog:** one shared list of problems, fixes and risk levels that both the training code and the app read

I built it in milestones with Claude Code, and each one got a spec agreed before any code was written. The [write-up](https://github.com/rishikeshvk/unstuk/blob/main/docs/writeup.md) has the full numbers, the charts and every mistake the model makes.
