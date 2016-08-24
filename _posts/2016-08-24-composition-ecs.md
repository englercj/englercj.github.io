---
layout: post_simple
title:  Fae and ECS
date:   2016-08-24 12:00:00

tags:
- fae
- programming
- javascript
---

TL;DR: I made an [ecs library][fae-ecs] that focuses on assemblages (composed entities), and treats
components as JS Mixins.

## Why build Fae?

Recently I've been working on building [Fae][fae], a modular 2D renderer. My main driving force for
building the project was exploring the question "If I was building Pixi.js, from scratch, today; what
would I do?"

<!--more-->

I thought about this for quite some time, and read as much as I could from users and library authors
talking about what they liked and disliked. One particular thing that stood out to me, is that over
the years pixi has gotten pretty big. It has a lot of features, and is very powerful; but most
people don't need (or want) all those features.

This is a particularly sticky issue for library authors. Many times they want sprites, but have their
own interaction/ticker/animations/etc. There are often pieces of the overall library that people just
don't want and with pixi taking those apart is pretty difficult due to some early design decisions we
made.

Because of this I wanted to start [Fae][fae] immediately with the idea that anything can be ripped out
of the library at anytime, and if it is, you pay no cost for it. I wanted to try and borrow the C++
mantra of "Don't pay for what you don't use."

## Extreme Modularity

One of the reasons it was hard for me to really separate out features in Pixi was because the library
is structured around a classical-inheritance-based mentality it inherited from Flash. This works great
for building a framework to use, but hinders trying to not have a component.

To give an example scenario, imagine your project doesn't use filters at all. Because of that you want
to exclude them from your build of pixi to save space and CPU for other fun things. To do this, you can
run the build with a flag to exclude filters. Problem solved! Well, kind of. This will exclude the
FilterManager, Filter, and other classes which results in file-size savings. But all the rendering checks
to try and render filters is still there, because that code is in Container. We hide a lot of this cost
behind if-statements and other things but there is always that `.filters` property on DisplayObject that
will still be there no matter what.

To fix this case I decided the only way to truly be modular was if I could exclude filters and the library
contained no code related to filters anywhere at all. For that to work, I would have to abandon the notion
of using classical inheritance as a basis for composing objects and instead deal with something more flexible.

## Entity-Component-System Pattern

ECS is getting more and more common nowadays, and for good reason. In native languages using
components as storage can increase cache hits due to data-locality, and objects can be modified
without affecting other similar objects. For me, it was a way to build a "Sprite" without knowing
all the features that sprite will have.

I started out toying with a lot of different ECS implementations and found a few that I liked. But,
once I got it all built using [yagl/ecs][yagl-ecs] I realized it was actually pretty combersome for
the user to have to compose an object everytime they wanted a Sprite. Additionally, having a factory
that automatically composed for you, really didn't make me feel any cleaner about what I was doing.

I really liked the Components and the Systems as described in ECS and implemented in these libraries,
but I didn't like Entities. I missed the days of just saying `new Sprite()`. That was always super easy.

## Components as Mixins

Just when I was losing faith in ECS, I found this article: ["Real" Mixins with JavaScript Classes][es6-mixin].
I highly recommend you read it, but the gist was instead of mixing properties into an object,
put the mixin on the prototype-chain. That seems like a simple concept, but it is extremely
powerful if you think it through. It solves a lot of the problems I encountered using mixins
before.

It also got me thinking, what if components weren't just dumb data containers. What if a component
was a mixin? What if an Entity wasn't an ID into a list of components, but a class that represented
a unique mix of components?

Using this slightly different definition of ECS I forked [yagl/ecs][yagl-ecs] and created [@fae/ecs][fae-ecs].
This new lib uses the "subclass factory" idea from the aforementioned article as Components.

Here is how it can look to use [@fae/ecs][fae-ecs]:

```js
import ECS from '@fae/ecs';

// a component is a "subclass factory".
const PositionComponent = (Base) => class extends Base {
    constructor() {
        // entities may have ctor args, so always pass
        // along ctor args in a component.
        super(...arguments);

        this._x = 0;
        this._y = 0;
    }

    get x() { return this._x; }
    set x(v) { this._x = Math.floor(v); }

    get y() { return this._y; }
    set y(v) { this._y = Math.floor(v); }
}

const TextureComponent = (Base) => class extends Base {
    constructor() {
        super(...arguments); // pass along ctor args

        this.texture = new Image();
    }
}

// an entity is a class that extends Entity, composed with as many components as you want
class Sprite extends ECS.Entity.with(PositionComponent, TextureComponent) {
    constructor(imageUrl) {
        super();

        this.texture.src = imageUrl;
    }
}
```

Now we end up with a `Sprite` class, just like we are used to. But if I as a user wanted a sprite
with no position, no problem:

```js
class MySprite extends ECS.Entity.with(TextureComponent) {

}
```

So Fae's job now becomes building components, and systems that operate on them. It also provides
some default assemblages (like the `Sprite` class above), but you don't have to use them. Or even
cooler, you can easily add to them.

Lets create a FilteredSprite:

```js
class FilteredSprite extends Sprite.with(FilterComponent) {

}
```

I can use the built-in sprite but extend it with a FilterComponent to get the feature-rich sprite
I want. As a real-world example, here is the sprite class in [Fae][fae]:

```js
import { ecs } from '@fae/core';
import { TransformComponent } from '@fae/transform';
import { TextureComponent } from '@fae/textures';
import SpriteComponent from './SpriteComponent';

export default class Sprite extends ecs.Entity.with(
    ecs.VisibilityComponent,    // whether or not to render
    TransformComponent,         // where to render
    TextureComponent,           // what to render
    SpriteComponent             // how to render
)
{
}
```

Then later in an example I create a `Card` that moves aroun and does stuff:

```js
const CardComponent = (base) => class extends base {
    constructor() {
        super(...arguments);
        this.speedX = 0;
        this.speedY = 0;
    }
};

class Card extends Fae.sprites.Sprite.with(CardComponent) {}
```

That final result is a sprite with some speed properties so I can move it around. Those who are
reading closely may be asking "Well, how does it move around?". That brings me to the last part
of ECS, the Systems.

## Efficient Systems

One of the main reasons I chose [yagl/ecs][yagl-ecs] as a starting point was due to how it handled
systems. It pre-calculates the systems an entity is eligible for when it is added to the system. Then
it will recalculate that if components on an entity change. It then iterates each system, and each
system updates it's own entities.

This is a pretty big difference from other libraries that iterate each system and tell it to update,
then the system (each update) queries eligible entities and operates on them.

In [@fae/ecs][fae-ecs] I tried to be even more clever. Since it is based around assemblages, the
components of an entity once created *cannot change*. That gives me a slight advantage in that
the only time I ever need to calculate eligibility is on add. Then I changed one other thing as
well. Instead of iterating the systems, each updating every entity, I iterate the entities and
tell each system to operate on it.

It is necessary to do it this way in order to have z-ordering. If I iterated each system, the order
of systems would dictate render order. For example, if I have a `SpriteRenderSystem` and a
`MeshRenderSystem` added to the ECS in that order, meshes would always draw on top of sprites.
By iterating entities and applying all eligible systems to them in series, you can have a sprite
then a mesh then another sprite on top. It also allows me to have a flat array ordered by z-index.

## Conclusion

I think I've landed on a pretty elegant way of reaching the goal I had set out to do. Each plugin
in Fae is really just one or more components and systems that a user can cobble together to create
the thing that they need. If they don't want filters, or bounds calculations, or whatever. They just
don't compose their entity with it (and build Fae without it for file-size savings).

I'd love to hear what others think about this approach, so leave comments below or hit me up
on Twitter ([@rolnaaba](https://twitter.com/rolnaaba)).

[fae]: https://github.com/Fae/fae
[fae-ecs]: https://github.com/Fae/ecs
[yagl-ecs]: https://github.com/yagl/ecs
[es6-mixin]: http://justinfagnani.com/2015/12/21/real-mixins-with-javascript-classes/
