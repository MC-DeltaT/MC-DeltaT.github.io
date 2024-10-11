---
title: The Photo Editing Software Dilemma
---

## Introduction

After taking photos as a hobby for over a year now, I have come to the unfortunate conclusion that editing software for photographers is pretty suboptimal. "But there are so many photo editors out there," I hear you say. Yes, there is a substantial quantity of photo editing software - yet no individual piece of software incorporates enough positive characteristics to be a clear winner in my eyes.

This article is my rants and thoughts about the state of photo editing software.

## Current Options

Let's take a look at some common photo editing software which I have tried out and been unsatisfied with.

### Adobe Lightroom

If I had to pick one software closest to being a winner, it would be Lightroom. The workflow is great (photo browser, global adjustments, mask adjustments, nondestructive), UI is good, performance is acceptable, and there are enough features for general editing. In my opinion, Lightroom finds a decent balance between simplicity and complexity.  
There's a huge "but", though: it's Adobe software. I think it's not controversial to say Adobe is one of those companies we could do without supporting. The subscription pricing model is quite consumer-unfriendly. Professional photographers might be able to suck up the ongoing costs, but for a hobbyist, Lightroom is not really cheap.

### Affinity Photo

This pick comes up often when asking around for a Lightroom alternative. The main attraction touted by its supporters is the option to buy the software outright with a one-time purchase (which is decently cheaper than a one year subscription to Lightroom). Like Lightroom, the UI is polished and the performance is great. Unfortunately, for me it's more like an alternative to Photoshop than Lightroom. There are limited capabilities for toggleable edits and no bulk editing support. Toggleable edits are a make-or-break for me, because I like to iterate through adjustments until I'm happy, which is tedious with only undo/redo.  
As much as I want to love Affinity, it's not what I, and probably other photographers, need for a general photo editing workflow.

### RawTherapee

RawTherapee is the first photo editor I used, in attempts to avoid paying for Lightroom. It's open source, which I love supporting. I managed with this editor for several months before eventually becoming dissatisfied with the somewhat limited colour controls and masking flexibility. RawTherapee is an solid tool for photographers who don't make a lot of edits to their photos.

### Darktable

On the other end of the spectrum of features is Darktable. It's a surprisingly capable open source editor, with plenty of fine-tunable lighting, colour, and detail adjustments, plus powerful masking. Although it has one problem disappointingly common in open source: the usability is lacking. Specifically, the performance is bad. Really bad on my machine, to the point of being intolerable (think seconds for each UI interaction).  
Respect to the Darktable developers for making a free product with such great features. Unfortunately, I can't justify buying a more powerful computer just to edit some photos. [^1]

[^1]: I would like to stress the point that I am not at all complaining about free software. The world is surely a better place for having software like Darktable available. Simultaneously, the reality of the situation is Darktable is unusable on my computer.

### Ansel

A relatively new contender, Ansel is a fork of Darktable to fix the aforementioned usability issues. It's designed to be a "cleanup" of Darktable to streamline the editing process and consolidate adjustments' colour science foundations. Ansel has been my editor of choice for the past several months, through hundreds of shots, and my growing frustrations with it have prompted this article. Because alas, as of mid 2024 it is alpha software, and still inherits many problems from Darktable. The performance is unchanged. And as is the nature of alpha software, there are bugs, possibly more than Darktable.  
I hope that Ansel grows to be the great editing software it aims to be.

## What Do I Want in an Editor?

So now that I've said a lot about what I don't want in photo editing software, what do I want?

Well, let's start with a list of core image adjustments I think every editor should have:

- Exposure
- Contrast
- Saturation
- "Vibrance" (i.e. saturation boost)
- RGB curves
- Tint by shadow/mids/highs
- HSL curves
- Colour LUT
- Dehaze
- "Clarity" (i.e. mid-detail contrast)
- "Texture" (i.e. fine-detail contrast)
- Crop
- Rotation
- Denoise
- Clipping reconstruction
- Lens corrections
- Border
- Film grain

All adjustments should be nondestructive and toggleable.

Image adjustments should be available globally, and also available as local adjustments applying only to selected image regions. Regions should be defined by hand drawn masks, and it would be nice to have the ability to refine them with pixel parameters (e.g. "all pixels above 0.9 luminance"). The workflow for local adjustments should be region-centric: regions should be defined in one place in the UI and be associated with multiple adjustments (unlike Darktable, in which regions are tediously defined on a per-adjustment basis).

I don't think it's necessary to allow for multiple instances of global adjustments, nor arbitrary manipulation of the order in which adjustments are applied. Darktable allows both of these, which I hypothesise could be a contributor to its poor performance. Maybe there is a case to be made for two layers of limited light and colour adjustments, for a base colour correction plus further colour grading.

A feature I love to see in image editors is a healthy dose of scopes and colour pickers. You can argue that editing by eye is the best, but scopes can be handy when you need a more quantitative approach. Vectorscope, RGB parade, and histogram should be standard. I was shocked to discover that Lightroom only has a histogram available, which I find to be the least useful of the three! Integrated into the scopes should be a pixel/colour picker which allows an unlimited number of markers.

The editor should have functionality to browse images in a folder, mark them as accepted/rejected, and copy edits across multiple images.

The performance should be no worse than Lightroom. That is, image adjustments should be applied in no more than 500ms on a quad core laptop CPU from 2017. I think this is quite a low bar to meet.

Finally, the software should be open source, or at worst, available for one-time purchase at a price <$100 AUD.

In my experience, no editor exists today which fulfils all of these requirements. Darktable and Ansel come close, but lack simple controls for certain adjustments, in addition to other mentioned usability problems.

## How Do We Get a Better Editor?

The features I want in a photo editor are well defined. How do I get them? This is, of course, the hardest part. There appear to be two options:

- Create a photo editor from scratch;
- Modify existing open source software to be what I want.

As a software developer, the first option feels fun and alluring. There is no technical reason why I cannot create the exact editing software I want in my vision. There are pratical reasons, though. I'm not great at creating GUI applications, and in general it's tricky to do a good job. Even with the UI aside, some image adjustments are surely complex and require legitimate research into colour science and signal processing. Creating a photo editor from scratch would be very time consuming for one person - time I currently don't have available to be consumed.

The second option of modifying existing software is probably more sane. At the same time, reading and tweaking code in a large project can be, in the worst case, almost as hard as rewriting from nothing. The closest software to what I want would be Darktable, and from what we see from Ansel, the Darktable codebase may not be the nicest to change. Further, nonfunctional requirements like performance can be hard to address because they are affected by the overall code design.

Perhaps there is a plausible hybrid approach, whereby I start from scratch but use existing open source code as reference for certain hard aspects of image processing. This approach could minimise reimplementation of the wheel and minimise time spent understanding existing systems.

## Final Words

Am I going to embark on a journey to bring photo editing software up to my standards? I'm not sure. It's possible doable and definitely tempting... Either way, it has been useful to reflect on my photo editing workflow.  
If you also edit photos, let me know your thoughts! Are you happy with existing software? What improvements would you like to see in this space?

## Footnotes
