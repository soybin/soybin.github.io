---
layout: post
title: raymarch the geometry & pathtrace the scene 
---

<!---[*fractal rendered with idyll using raymarching and path tracing*](/images/idyll/first.png){:width="100%"}--->
# {{ page.title }}  

## preamble
Today, I'll superficially explain and reflect on raymarching and pathtracing; the two main algorithms behind the rendering pipeline that I came up with when developing  [**_idyll_**](https://www.youtube.com/watch?v=cFykbtJmg4A), a fractal engine.  

## definitions
* *Ray*: simple structure composed of two \\(n\\) dimensional vectors: an **origin** vector, which indicates at which point in space the ray originates, and a unit **direction** vector, which denotes the direction in which the ray will move.  
   
* *Geometry*: any independent model, shape or object present in the scene. 
  
* *Scene*: \\(n\\) dimensional space in which the geometry is represented.

## raymarching
Raymarching is basically a raycasting-based algorithm used to interpret a scene.
  
The algorithm itself doesn't compute lighting, shadowing nor reflections on its own, it rather gives you information about whether a ray emitted from a pixel does or doesn't intersect the surface of any geometry in the scene. In other words, it calculates the depth of the scene.
  
### distance estimators
Imagine that you're working on a 3-dimensional space, *in order to perform raymarching, you need a function that tells you how far away is any given point in space from the closest point to the surface of the geometry*. That's called a __*distance estimator*__, DE from now on.  

The most basic DE functions represent primitive objects, and [**_this_**](https://www.iquilezles.org/www/articles/distfunctions/distfunctions.htm) is a fantastic resource by Íñigo Quílez on them.  
  
For example, here you can see the DE function for a sphere \\(sphereDE\\), where \\(P\\) is the point in space that you're calculating how far apart from the sphere it is, and \\(S\\) is the scale of the sphere (assuming the sphere is located at the origin of coordinates):

\\[ sphereDE(P, S) = length(P) - S\\]
```
double sphereDE(vector3 P, double S) {
	return length(P) - S;
}
```
### what to do with a DE

**_Once you know how far away from something you are, you know you can safely move that distance in any direction, without having to worry about hitting that object._**  
  
* Let a camera be placed anywhere in your scene with a distance of 10 meters from the origin of coordinates (0, 0, 0), and looking towards it.  
  
* Let a ray have its origin (ori) at the position of the camera in your scene, and give it a direction (dir) towards the center of coordinates.  
  
* Let a variable total distance (tDist) represent how far away has the ray moved from its origin, and it'll be initialized to zero.  
  
* Now, calculate the DE from the point O + D * D' (the origin of the ray plus the direction of the ray times the distance the ray moved).  
  
* Then you can safely *__march__* the ray by the distance returned by the DE function, which means you can add DE(O + D * D') to D.  
  
* Repeat this process until DE(ori + dir * tDist) returns a value smaller than a certain arbitrary threshold (1e-4 for example), which means that the ray is close enough to the surface of the geometry for it to be considered an intersection. Obviously the ray won't get to hit the geometry itself because of Zeno's paradox, so it's an estimation, hence the name distance estimator.

This process should be computed for each pixel on the screen, and it'll provide an accurate interpretation of the model, no matter how complex the distance estimator may be.

The pseudocode for the simplest version of this algorithm would look something like the following:
```
// distance threshold
const double MIN_DISTANCE = 1e-4

/*
 * the maximum number of steps allowed. 
 * if this amount of steps, 'raymarch'
 * will return 0.0
 */
const int MAX_STEP = 64
 
color raymarch(vector3 ori, vector3 dir) {
	double tDist = 0.0
	int i
	for (i = 0; i < MAX_STEP; ++i) {
		double curDist = sphereDE(ori + dir * tDist)
		// in case the threshold is reached
		if (curDist < MIN_DISTANCE) {
			break
		}
		// else, add the marched distance
		// to the distance variable
		tDist += curDist
	}
	return color(1.0 - (double)i / (double)MAX_STEP)
}
```

This is a very simple implementation, and if you were to use the returned value from this function as the value for the three color channels of each pixel, you would get a black and white image with some very obvious banding artifacts (since we're deriving the return value from two integers).  
  
There's also a lot of things I've left out of this explanation because they're beyond the scope of this article, such as primitive shapes transformation and combination (somewhat similar to intersection boolean operations), spatial transforms, and even [infinite repetition](https://www.youtube.com/watch?v=zy1l1SLJDo4), among many others. Minimally modifying the DE for your geometry will dramatically alter the end result of the geometry.

### ray direction
I've skipped over this until now, because it's not vital to conceptually understanding raymarching. However, it is a very important implementation detail in most cases. The following, is an example of a formula for calculating the direction that each pixel's ray \\(rayDirection(P_x, P_y)\\) should follow, given a desired field of view \\(fov\\) in radians:  
  
Let the constant \\(P_x\\) be equal to the pixel x-coordinate on your screen.  
Let the constant \\(P_y\\) be equal to the pixel y-coordinate on your screen.  
Let the constant \\(width\\) be equal to your screen's width in pixels.  
Let the constant \\(height\\) be equal to your screen's height in pixels.  
  
\\[rayDirection(P_x, P_y) = [x, y, -z]\\]
\\[x = P_x + 0.5 - \frac{width}{2}\\]
\\[y = P_y + 0.5 - \frac{height}{2}\\]
\\[z = \frac{height}{tan(\frac{fov}{2})}\\]
  
This formula applies to most physically based rendering (pbr) techniques, and the implementation itself is very straighforward.

### real world usage
As long as you can derive a function that, for any given point in 3d space, returns its distance from the closest point to any geometry in the scene, you can render pretty much anything with a great performance. Adding dynamic lighting, shadowing, occlusion and surface color would have a very low computational cost. Even changing the geometry's scale, position and rotation. Anything is valid, because everything is dynamic and computed in real time.  
  
Raymarching is not mainstream in the videogame industry, the reason for this is that you __*need*__ a DE function, and in order for it to run in real time, all of it should to be computed in a fragment shader on the GPU side. When you have a world with hundreds of independent entities interacting with each other, materials to be computed, and very specific and complex geometrical shapes, it becomes counterproductive to have an efficient DE.  
  
On the other hand, whenever you have an accurate DE, you can efficiently compute your scene using raymarching. This makes raymarching the ideal method to render complex mathematical models, such as fractals or combinations of these.  

## pathtracing
Pathtracing is a rendering technique derived from raytracing. We'll discuss the differences between this two terms later in this article.  
  
There are two important constants to define when implementing a pathtracing algorithm: the samples per pixel, and the bounces per ray.
  
The idea behind pathtracing is to *__iteratively integrate most of the light arriving to a point in the surface of the geometry__*, so that the image looks as real as possible.
To visualize it, this is the pseudocode for a very basic pathtracing algorithm (only Lambertian reflection):
```
// number of samples per pixel
const int SAMPLES = 256

// maximum number of allowed ray bounces
const double BOUNCES = 8;

// probability of ray reflection
const double PROB = 1/PI

// pathtracing recursively
color pathtrace (ray r, int bounces) {
	// in the ray missed
	if (!r.hit) return color(0.0)

	// in case we reached max number of bounces
	if (bounces == BOUNCES) return color(0.0)

	// get ray normal
	vector3 rayNormal = calculateNormal(r.origin, r.direction)

	// get current new ray origin
	vector3 newOrigin = r.origin + r.length * r.direction;

	// get random unit vector in the hemisphere
	// of the normal, so that the ray can bounce off
	vector3 newDir = getHemisphereRandom(rayNormal)

	// get color at the intersected point
	color color = getWorldColor(r.origin)

	// bidirectional reflectance distribution function
	double ct = dot(newDir, rayNormal)
	color brdf = r.hitReflectance / PI

	ray newRay = {newOrigin, newDir}

	color nextBounce = pathtrace(newRay, bounces + 1)

	// this is where the integration of
	// the lighting along the different
	// intersection points of the ray
	// happens
	return color + (brdf * nextBounce * ct / PROB)
}

// iterate through every pixel in the screen
for (y in height) {
	for (x in width) {
		// get ray's direction and origin
		vector3 direction = rayDirection(x, y)
		vector3 origin = {0, 0, 10.0}
		// variable storing color
		color totalColor = {0.0, 0.0, 0.0}
		// iterate for every sample and add to color
		for (sample in SAMPLES) {
			totalColor += pathtrace(origin, direction, 0)
		}
	}
	color /= samples
	pixels[y][x] = color
}
```
What's happening here is the following:
* Iterate through every pixel in the screen or image.
* ~ Compute the ray's origin and direction from the pixel coordinates.
* ~ Iterate through every sample.
* ~ ~ Add the return value of the function 'pathtrace', which randomly computes a possible value for that ray, to the 'totalColor' variable.
* ~ Divide the 'totalColor' variable by the number of samples to get the average value for all of the samples.
* ~ Set the pixel with 'x' and 'y' coordinates to the 'totalColor' variable value.  

With a high enough value for the samples and bounces variable, you can get very photorealistic images using path tracing.
## disambiguation

If you're reading this, chances are you've heard of raytracing, and it's no coincidence raytracing, raymarching and pathtracing sound familiar.  
  
All three of these PBR (physically based rendering) algorithms have something in common: *__they all use raycasting to understand a scene's geometry__*, but what each one of this algorithms accomplishes, and the way they operate is different from one another. It's also true that most researchers have slightly different understandings of these concepts, since the lines separating them are often blurry.
I'll try to shed some light regarding the differences between these techniques based on my own experience and understanding.

### raymarching and raytracing
In raymarching, the intersection with the scene's geometry is computed using a distance estimator, as we've recently seen. On the other hand, raytracing computes the intersection analitically, not requiring a DE. Coming up with a DE for a fractal or any mathematical model can be easy, but when you have many objects with different shapes, raymarching grows exponentially in price. This makes raytracing―although still pretty expensive―more suitable for games, and it's also the reason why raytracing isn't, and probably won't be, the mainstream rendering technique.  
That being said, raymarching is awesome; as I mentioned earlier, when used properly, it can deliver unprecedented performance on very specific applications, and its simplicity makes it very straightforward to understand, implement, and maintain.  

### pathtracing and raytracing
Pathtracing is a physically based rendering (PBR) technique that relies on the monte carlo method. It integrates the amount of light arriving to a specific point in a surface of an object in the scene by averaging out different paths that the ray could take when bouncing off that surface. That's the overall idea, but whether that point be the camera lenses or an object's surface depends on the specific implementation.  
In the case of **_idyll_**, each point of intersection from a camera ray intersecting the geometry is pathtraced to compute the amount of light arriving to that point; a combination of pathtracing and raytring.  

## union of raymarching and pathtracing
Now you can see how raymarching and pathtracing can be complementary to each other, making for a solid rendering pipeline:  
With raymarching, you can check whether a ray intersects the geometry and where. Then, you can use pathtracing to integrate the global illumination arriving at the point of intersection between the surface of the geometry and the marched ray.  
   
In *__idyll__*, pathtracing computes the global lighting, but doesn't compute the point of intersection with the geometry. On the other hand, raymarching computes the point of intersection with the geometry, but doesn't compute lighting. This is an oversimplification, as the actual relation between the two techniques is a bit more entangled, but you get the overall picture.
## results
After properly implementing these techniques, and throwing some iterative fractals into the mix, I developed [**_idyll_**](https://github.com/soybin/idyll), a fractal engine that can create beautiful renders like the following:  

![*fractal rendered with idyll using raymarching and path tracing*](/images/idyll/first.png){:width="100%"}
![*fractal rendered with idyll using raymarching and path tracing*](/images/idyll/second.png){:width="100%"}
![*fractal rendered with idyll using raymarching and path tracing*](/images/idyll/third.png){:width="100%"}
