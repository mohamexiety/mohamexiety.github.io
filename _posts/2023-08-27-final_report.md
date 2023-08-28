---
title: NVK YCbCr Support - GSoC 2023 Final Report
date: 2023-08-27 18:00:00 +/-0000
categories: [blog]
tags: [gsoc, nvidia, nvk, gpu]     # TAG names should always be lowercase
author: mohamexiety
toc: true
toc_sticky: true
---

## Introduction

Hello! This is the final report of the work I did as a Google Summer of Code 2023 contributor to NVK. My work revolved around the implementation of YCbCr format support, which came in form of enabling three Vulkan extensions.

Mesa is the open-source, default graphics driver stack on Linux, with implementations for graphics hardware from most vendors. One such implementation is NVK, a new driver which is on its way to being the default implementation when shipping software targeting NVIDIA graphics hardware on Linux. In other words, this means that it will inevitably become the default driver across installations, unless users actively seek out the properietary NVIDIA driver stack. Due to this, not only does NVK need to support all features and extensions, it needs to do so in a clean, modular way that can be supported across hundreds of different target architectures, performance tiers, and device integrations. In addition to that, NVK is also being used as the new basis for Vulkan drivers across Mesa, which means that it will serve as the baseline implementation for graphics implementations targeting NVIDIA, AMD, Intel, Apple, Adreno, Mali and other GPU families.

While most folks typically think of images in terms of the commonly known RGB mixed luminance and chroma encoding, there exists another type of image format encoded as luminance and separate chroma images, known as YCbCr.

![ycbcr](/assets/nvk_ycbcr/ycbcr_illust.png){: w="800" h="275" }
_YCbCr versus RGB. Image taken from Wikipedia, created by LionDoc - Own work, published in the Public Domain_

YCbCr is a color representation in which color information is encoded in terms of a luminance component (the 'Y'), and color difference components ('Cb' being blue difference, and 'Cr' being red difference). While on the surface it may seem as just another format or encoding, it's actually quite important in many fields such as media; this is because our eyes are more sensitive to changes in luminance over changes in color. Consider how the difference between ‘light’ and ‘dark’ modes tends to be rather vast and perceptible to most (if not all), yet choosing to paint a wall taupe or sandstone can be something couples will argue about until the end of time—therein lies the power of YCbCr. Once one has luminance separated away from color difference, they can heavily compress the aforementioned color difference components, thus leaving the luminance component as is. Doing so will sacrifice virtually nothing in perceptible visual quality, all the while saving a significant amount on bandwidth. It follows then that YCbCr formats are widely used in media, so the Vulkan API needs to support these formats and offer a convenient interface to manipulate data encoded in this format, which in turn means that the driver must offer a way for the API to interface with the hardware with YCbCr encoded data.

## The Project

At a high level without going into detail, the project mainly consisted of the following pull requests:

- [nvk: Enable multiplane image and image view support](https://gitlab.freedesktop.org/nouveau/mesa/-/merge_requests/227)
- [Improve YCbCr Multiplane Format Support in vk_format.c](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/24096)
- [nvk: Add YCbCr Support](https://gitlab.freedesktop.org/nouveau/mesa/-/merge_requests/226/)
- [nvk: Add support for single-plane YUV formats](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/24614)

Additionaly, I maintained a blog detailing my progress as I went about the project. However, it's not complete yet, unfortunately, as I got busy and had to delay writing a bit:

- [NVK YCbCr Support Part 0 - Hello World](https://mohamexiety.github.io/posts/hello_world/)
- [NVK YCbCr Support Part 1 - Multiplane Support](https://mohamexiety.github.io/posts/multiplane/)
- [NVK YCbCr Support Part 2 - YCbCr Advertisement](https://mohamexiety.github.io/posts/format/)
- NVK YCbCr Support Part 3 - Sampler and Compiler Support
- NVK YCbCr Support Part 4 - Single-plane YCbCr Format Support

Going deeper and breaking things down further, YCbCr support consisted of around 5 phases as outlined below, in order of implementation:

### I. Support multi-plane images/image views

MR:

- [nvk: Enable multiplane image and image view support](https://gitlab.freedesktop.org/nouveau/mesa/-/merge_requests/227)

Blog post:

- [NVK YCbCr Support Part 1 - Multiplane Support](https://mohamexiety.github.io/posts/multiplane/)

This part was the main "meat" of the project, and the one with the most changes; it was essentially a refactor of all the image and image-view related code to add support for multi-plane images in the driver. The blog post linked above goes into things with much more detail, but the long and short of it is that multi-planar images are essentially images consisting of multiple (up to 3) images, with a few special rules around binding. This phase is necessary because many YCbCr formats are multi-planar themselves.

This part was its own MR and merged earlier before the rest of the project was finished because it was a relatively big refactor, and with NVK in active development, it was better from a project management perspective to merge it earlier rather than deal with merge conflicts later.

### II. Advertise YCbCr formats

Commits:

- [nouveau/nvk: Enable VK_KHR_sampler_ycbcr](https://gitlab.freedesktop.org/nouveau/mesa/-/merge_requests/226/diffs?commit_id=7445f1ef40c9b807186dab52e18c35ee7498cb29)

Blog post:

- [NVK YCbCr Support Part 2 - YCbCr Advertisement](https://mohamexiety.github.io/posts/format/)

This phase is concerned with advertising the extensions, and hence ultimately finalizing the project and enabling applications to use the Vulkan YCbCr features. However, despite that, it was the second phase implemented to facilitate testing and debugging.

### III. Compiler support for YCbCr samplers

Commits:

- [nouveau/nvk: Add YCbCr sampler NIR lowering pass](https://gitlab.freedesktop.org/nouveau/mesa/-/merge_requests/226/diffs?commit_id=b40c2798b452b419e7b4ea1fcb371ee5beabffcc)
- [nouveau/nvk: Support multi-plane descriptors in nvk_nir_lower_descriptors.c](https://gitlab.freedesktop.org/nouveau/mesa/-/merge_requests/226/diffs?commit_id=bbcad8b3074759d1a9c6a4b23cab8d5bcd099060)

Blog post:

- TBA

The third phase adds compiler support for YCbCr formats by adding in support for YCbCr samplers in the NIR optimization passes that the driver leverages for texture operations, as well as by refactoring a small bit of the same code in order to handle multi-plane descriptors properly. This phase enables the compiler to handle YCbCr samplers properly, and hence allows us to do sampling operations on YCbCr textures once sampler support is coded in in the next phase.

### IV. YCbCr sampler support

Commits:

- [nouveau/nvk: Create helper function for sampler creation](https://gitlab.freedesktop.org/nouveau/mesa/-/merge_requests/226/diffs?commit_id=b5e4d56f833d9b5e393fc2a676bafeb1f3251d90)
- [nouveau/nvk: Add multiple sampler planes for CONVERSION_SEPARATE_RECONSTRUCTION_FILTER_BIT](https://gitlab.freedesktop.org/nouveau/mesa/-/merge_requests/226/diffs?commit_id=7ca34eb23d627b46848bab59b300dcdfa8c27219)

Blog post:

- TBA

Building upon the first and third phases, this part adds YCbCr sampler support by modifying the existing sampler implementation to leverage existing common Mesa code for YCbCr conversion structures. Along the way, I also had to do a refactor to how samplers are handled and introduce multi-plane samplers to support `CONVERSION_SEPARATE_RECONSTRUCTION_FILTER_BIT`, which allows the use of different filters between the luminance and chroma planes.

### V. Single-plane YCbCr format support

MR:

- [nvk: Add support for single-plane YUV formats](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/24614)

Blog post:

- TBA

The final phase adds support for single-plane YCbCr formats by adding in the new formats to the driver and then wiring them to the internal hardware color formats.

## Thoughts and Experience

Overall, I'd say this project was a big milestone for me; it's my first foray into open-source, my first large-scale software project (_and_ my first shipped software!), my first driver development experience, and I am truly happy about it all and how it turned out. I admit that when I first began, it was rather unnerving and I was worried I would not be up to it, but in the end it worked out very well.

I ended up getting to know and deal with some really awesome people, learnt a lot more about software and programming like the things that go into refactors of large code bases, or the design decisions and tradeoffs that go into implementing features or carrying out certain changes in large-scale systems. I also learnt a lot about how GPU driver stacks work, and how the pieces fall into place into getting things to work in the end. Not just that, but the differences between what APIs expose to a programmer, and how the hardware actually functions and the driver heroics that go into making things as transparent as possible to the programmer. And in similar vein, as someone from a primary hardware design background, I learnt quite a bit regarding (and saw examples of) hardware decisions and tradeoffs influencing software decisions and forcing particular tradeoffs. Finally, I am mostly self-taught in my software skills, and completely self-taught in anything related to graphics, so this project has been a _boon_ in filling the gaps and holes in my knowledge, and making many things fall into place, so to speak.

For things I am disappointed about and wish I had done better, the main thing that comes to mind is that I wish I could have been able to manage my time a bit better so as to blog more often, as well as asking earlier about core concepts rather than stubbornly trying to brute force figuring out everything on my own, which often led to incomplete/improper understanding, and sometimes I would even fail at figuring out things in the end. I got better at this, but a bit later than I would have liked.

## Future Work

While thankfully I managed to finish the project, and hit all goals and targets so that there's nothing else to do in this area, I enjoyed it immensely and wanted to contribute even more and work on something else. I asked around, and I picked up both a relatively small [task](https://gitlab.freedesktop.org/mesa/mesa/-/issues/9525), and another [project](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/24640), which I will start working on these days. The task deals with enabling midpoint sampling for single-plane YCbCr formats on Intel and NVIDIA hardware (which default to even sampling) by modifying the NIR pass that deals with sampling. The project involves adding support for linear images on NVK, which are currently unsupported.

## Acknowledgements

I would like to express my deepest thanks and gratitude to my mentor, Faith Ekstrand, who helped _a lot_ throughout the whole project; she was very helpful, would always offer insight and advice whenever I needed it, and was all around just awesome to work with. In addition, special thanks goes to the X.Org admin for their trust in accepting my proposal, and Google and the Google Summer of Code admins for offering this initiative.
