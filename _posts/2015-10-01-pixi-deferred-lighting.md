---
layout: post_image
title:  Deferred Lighting with Pixi.js
date:   2015-10-01 12:00:00

image_file: blog/pixi-lights.jpg

tags:
- pixi.js
---

## Introduction

You can find the pixi-lights plugin [on github][code].

Some of you may be aware that I recently did an experiment with Pixi.js to see if I could write efficient normal-mapped lighting.
Well I had no idea where to start, how to do lighting, or what the best way to do it would be. So, I did a bunch of research on how
drawing normal-mapped lighting works, and came up with a way to get it drawing correctly using this tutorial (thanks Matt DesLauriers):

https://github.com/mattdesl/lwjgl-basics/wiki/ShaderLesson6#IlluminationModel

It was pretty dirty. Basically I wrote a filter that did N-lighting that was forward-rendered. For those who don't know, the normal
renderer in Pixi.js is a *forward-renderer*. That means it iterates "forward" through the scene list and draws each object entirely
as it goes along. Anyway, the idea behind the filter I wrote was that by adding this filter to a container it would add lighting to
that container. Problem was that since it was forward rendered each light had to be shaded for each object in the container. In the
few projects I wanted to use the effect in, it wasn't performant enough for the amount of lights I wanted to have. So I started
looking for a better way.

## Deferred Rendering

Eventually I found the concept of deferred rendering. We will talk more about how it works, but the goal of deferred rendering is to
*defer* the lighting until the scene is done drawing. With forward-rendered lighting you draw an object, then light it, then go to the
next object. What that means is that the number of draws you have are `(numLights * numObjectDraws)`. With lighting in a deferred-renderer
you draw all the objects, then apply lighting to them all at once. What that means is that the number of draws you have are `(numLights + numObjectDraws)`.
So the efficiency of your rendering increases linearly instead of exponentially. As you add more and more lights that fact becomes more
and more important.

### Challenge #1: I knew nothing about deferred rendering

The first challenge was just learning how deferred rendering works, and how it is implemented in other rendering systems. One of
the most useful articles I read was by Casey Carlin with Whole Hog Games:

http://www.wholehog-games.com/devblog/2013/06/07/lighting-in-a-2d-game/

I highly recommend reading it, but to summarize it was a great explanation of how deferred rendering works in 2D. The basic
methodology involves rendering different representations of objects to multiple output framebuffers, then combining that information
in a pass of lighting to output the final rendered view. It looks something like this:

![gbuffer](/img/blog/deferred_overview.png)

Like you can see in the image above, objects draw themselves to multiple render targets. Each render target holds information about the
object. One each for normals, depth, specular, diffuse, self-illumination, and anything else your lighting algorithms will need. This
series of framebuffers are often referred to as the G-Buffer. The trick that makes this mechasim performant is the ability to draw to
multiple render targets in a single shader simultaneously. The code can look a little like this:

```glsl
#extension GL_EXT_draw_buffers : require
precision mediump float;
void main() {
    gl_FragData[0] = vec4(1.0, 0.0, 0.0, 1.0);
    gl_FragData[1] = vec4(0.0, 1.0, 0.0, 1.0);
    gl_FragData[2] = vec4(0.0, 0.0, 1.0, 1.0);
    gl_FragData[3] = vec4(1.0, 1.0, 1.0, 1.0);
}
```

Note the first line of the shader there, `#extension GL_EXT_draw_buffers : require`. WebGL 1.0 (OpenGL ES 2.0) doesn't have support
for Multiple Render Targets (MRT). Support for MRT is implemented in WebGL 2.0 (OpenGL ES 3.0), and exposed in WebGL 1 via the
extension shown above. Unfortunately the support penatration for the extension and/or WebGL is really low. According to
[WebGL Stats][wglstats] less than 50% of the visitors to the monitored sites have browsers that suport the `GL_EXT_draw_buffers`
extension. I knew that if I wanted the plugin to have any kind of usage I couldn't use this feature. Which brings us to challenge #2.

### Challenge #2: WebGL has no MRT support

How do I draw the multiple framebuffers of data that my lighting shaders will need without support for MRTs? The answer is, unfortunately,
to draw each of the buffers in sequence as separate passes. To reduce the number of render passes I decided to only use the diffuse
and normal buffers. We get a pretty good effect and it only will take 3 passes (diffuse pass, normal pass, lighting pass). In 2D
these buffers are sufficient to give some good effects.

Unfortunately this means that for simple applications with only a few lights, the renderer I built will likely be much slower than
the normal renderer in pixi. Once you get to a complex application, with many lights, that is when you will see the performance
benefits of using this deferred renderer over an implementation in the forward renderer.

## Implementation

Alright, that is enough prologue, let's look at some code.

## The Future

For this plugin to be something that makes sense to use in "real projects" it needs to have robust support for self-illuminating objects,
and potentially specular mapping. In order to do those things I will need MRT support. This likely means rewritting the plugin to
only work on browsers with either `GL_EXT_draw_buffers` or WebGL 2 support. When I write v2 of this plugin it will likely have
those restrictions, and worked best for packaged apps and other controlled environments.

[code]: https://github.com/pixijs/pixi-lights
[wglstats]: http://webglstats.com/
