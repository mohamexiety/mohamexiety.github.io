---
title: NVK YCbCr Support Part 0 - Hello World
date: 2023-06-16 08:00:00 +/-0000
categories: [blog]
tags: [gsoc, nvidia, nvk, gpu]     # TAG names should always be lowercase
author: mohamexiety
toc: true
toc_sticky: true
---

# Introduction

Hello! I am Mohamexiety, a (soon-to-be) 4th year electronics engineering student and a recently accepted contributor in the Google Summer of Code program with my project being implementing YCbCr support for NVK, the new open-source nouveau Vulkan driver, under the mentorship and supervision of Faith Ekstrand [@gfxstrand](https://mastodon.gamedev.place/@gfxstrand) for the X.Org foundation. As part of the project requirements, I am to blog my progress and changes as I work and so I started this small blog. This project is honestly one of the things I am quite excited and passionate about as it's a huge learning experience for me to say the least, and is basically the biggest I have worked on so far, so now... let's begin. 

# Why YCbCr?

YCbCr is a color representation where color information is encoded in terms of a luminance component (the 'Y'), and color difference components ('Cb' being blue difference, and 'Cr' being red difference). While on the surface it may seem as just another format or encoding, it actually has a very important usage. As it turns out, our eyes are more sensitive to changes in luminance over changes in color, so when you have luminance separated away from color difference, you can heavily compress the color difference components and leave the luminance component as is and sacrifice nearly nothing in perceptible visual quality, while saving _massively_ on bandwdith.

## But Why Driver Support?

The simplest answer to this is that it's part of the core Vulkan 1.1 specification, and hence it needs to be implemented. As for _why_ it's part of the specification, YCbCr formats are widely used in media; in fact, nearly all videos are 4:2:2 or 4:2:0, so the API needs to support these formats and offer a convenient interface to manipulate data encoded in this format. Naturally then, the driver must in turn offer a way for the API to interface with the hardware with YCbCr encoded data. 

# The Plan

YCbCr support, loosely speaking, consists of around 4 phases:

**1. Support multi-plane images/imageviews** 

The concept of multi-planar images is essentially where your single image can be up to 3 images; each "image" here representing a plane. Currently, the driver works on the assumption that all images are single-plane. This part is pretty big, in fact one of the biggest parts of this whole project, and it touches things all over the driver with rewrites to some parts. 

This is needed because the color formats in Vulkan for YCbCr can be encoded either as a single plane image, a 2-plane image, or a 3-plane image depending on the chosen format. The main idea here is having each of the channels separated into its own plane, or shared with another channel (or 2 other channels) in the same plane. 

For a practical demonstration, we can take a look at `VK_FORMAT_G8B8G8R8_422_UNORM`, which is a subsampled, single-plane format. Here we have 2 G components, an R component, and a B component. All of these encode a 2×1 rectangle RGB texel, where subsampling is done by having a G value present at each horizontal coordinate, but the B and R values are shared across both G values and so they are at half the horizontal resolution of the total image in the end. 

On the other end of the specturm, we have `VK_FORMAT_G8_B8_R8_3PLANE_422_UNORM` which essentially stands for the same thing above, but 3-plane. So here, we have a plane for the G component (plane 0), a plane for B (1), and a plane for R (2). Subsampling then is done by having the R and B component planes at half the horizontal resolution of the G component plane and the whole image.

And in between, there is `VK_FORMAT_G8_B8R8_2PLANE_422_UNORM`, the 2-plane version of the above. It's the same idea as above except the first plane is 8-bit G, while the second plane instead is 16-bit BR (B in byte 0, R in byte 1).

**2. Advertise YCbCr formats** 

After enabling multi-plane image support, we're now all set for advertising YCbCr color formats in the driver, which will allow applications that require these to work with them right away. The color formats mentioned here are of course all the subsampled and multisampled ones, along with a few uncompressed formats where the channels are encoded in multiple planes. There's thankfully a lot of helpful common code in mesa that will make this stage fairly easy and straightforward. However, we may end up having to do a little bit of reverse engineering later for subsampled images as the open-sourced NVIDIA header files are missing some data here.

**3. Add color conversion objects into nvk_sampler**

The Vulkan specification defines a few [conversion objects](https://registry.khronos.org/vulkan/specs/1.3-extensions/man/html/VkSamplerYcbcrConversion.html) as part of its YCbCr implementation which we'll need to implement and wire up to the NVK sampler. This is another mostly fairly straightforward phase, but there are a few complications with hooking things up to `nvk_sampler` which will make this fun.

**4. Compiler support for YCbCr samplers**  

I am honestly not sure what this will fully entail yet so don't have many details to put here for now. This will likely start with a copy-paste job from another driver, however. 

## Where Are We?

So out of all that, where exactly are we at right now? We're roughly 3/4 the way through step 1. More specifically, `nvk_image` now supports multi-plane properly, along with `nvk_image_view`, and finally, descriptor support for multi-plane is also halfway done. This leaves the remaining part of descriptors, and then the parts concerning drawing and copying images. However, it's important to note that this is untested as of yet so there will likely be some time to clean up bugs as well as clean up the code in general.   

# Conclusion and What Comes Next

That's about everything, I suppose. I have been working on this a fair bit earlier than this post but didn't have much time earlier to write sadly due to university pressure, so there is going to be another post coming up very soon detailing the process I went through to doing multi-plane so far. Given the scope of changes, multi-plane will likely be spread out across 2 posts or even more. After that we'll get to the other phases, where I'll be ideally posting every week or every two weeks depending on progress and challenges. Thanks for reading up to this point, and I hope the coming posts end up a good reading experience; I can't wait! 
