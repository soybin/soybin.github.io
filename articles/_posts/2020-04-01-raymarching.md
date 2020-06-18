---
layout: post
title: raymarching geometry & path-tracing the scene
---

<!---[*fractal rendered with idyll using raymarching and path tracing*](/images/idyll/first.png){:width="100%"}--->
# {{ page.title }}  
## prelude
Today, I'll explain and reflect on the logic behind the rendering pipeline that I came up with when developing  [**_idyll_**](https://www.youtube.com/watch?v=cFykbtJmg4A), a fractal engine.  
  
I'll skip over the implementation details, and I'll focus instead on the main concepts and ideas, since I can guarantee that the valuable learning experience is to implement these concepts yourself. However, checking the [**_source code for idyll_**](https://www.github.com/soybin/idyll) can help if you're feeling a bit lost.

## raymarching
### naming
Most people would refer to raymarching as a rendering technique.  
I wouldn't go as far as calling it a rendering technique on its own, since the raymarching algorithm itself doesn't compute lighting, reflection nor shadowing, by default. I would say it's more of a 'scene interpretation technique', if such thing even exists.  
  
Anyways, that was a bit of a tangent. The important thing for you is to know what raymarching conceptually implies, since, as far as I'm concerned, there's no formal definition, and you could apply it in various different ways.  
  
### distance estimators
Now, imagine that you're working on 3D space, *you need a function that tells you how far away from the closest point in the scene you are*. That's called a __*distance estimator*__, DE from now on.  

The most basic DE functions represent primitive objects, and [**_this_**](https://www.iquilezles.org/www/articles/distfunctions/distfunctions.htm) is a fantastic resource by Íñigo Quílez on them.  

For example, here you can see the DE function for a sphere \\(sphereDE\\), where \\(P\\) is the point in space that you're calculating how far apart from the sphere it is, and \\(S\\) is the scale of the sphere (assuming the sphere is located at the origin of coordinates):

\\[ sphereDE(P, S) = length(P) - S\\]

### what to do with a DE

**_Once you know how far away from something you are, you know you can safely move that distance in any direction, without having to worry about hitting that object._**  
  
* Let a camera (C) be placed anywhere in your scene with a distance of 10 meters from the origin of coordinates (0, 0, 0), and looking towards it.  
  
* Let a ray have its origin (O) at the position of the camera in your scene, and give it a direction (D) towards the center of coordinates.  
  
* A variable distance (D') will represent how far away has the ray moved from its origin, and it'll be initialized to zero.  
  
* Now, calculate the DE from the point O + D * D' (the origin of the ray plus the direction of the ray times the distance the ray moved).  
  
* You can safely *__march__* the ray by the distance returned by the DE function, which means you can add DE(O + D * D') to D.  
  
* Repeat this process until DE(O + D * D') returns a value smaller than a certain arbitrary threshold (1e-4 for example), which means that the ray is close enough to the surface of the geometry for it to be considered an intersection. Obviously the ray won't get to hit the geometry itself because of Zeno's paradox, so it's an estimation, hence the name distance estimator.

### reflection on raymarching

This means that as long as you can derive a function that, for any given point in 3d space, returns its distance from the closest point to any geometry in the scene, you can render pretty much anything with an impressive performance. Adding dynamic lighting, shadowing, occlusion and surface color would have a very low computational cost. Even changing the geometry's scale, position and rotation. Anything is valid, because everything is dynamic and computed in real time.  
  
It's also the reason why raymarching is not mainstream in the videogame industry: you __*need*__ a DE function, and in order for it to run in real time, all of it has to be computed in a fragment shader on the GPU side.  
  
When you have a world with hundreds of independent entities interacting with each other, materials to be computed, and very specific and complex geometrical shapes, it becomes impossible to have an efficient DE.  
  
On the other hand, when you can derive a DE function from your geometry, you can efficiently compute your scene using raymarching. This makes raymarching the ideal method to render fractals, which is what I ended up doing for idyll.

## distance estimating an iterative fractal

