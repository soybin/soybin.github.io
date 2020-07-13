---
layout: post
title: using computer vision to virtualize motion
---

# {{ page.title }}  

## preamble
Today, I'll reflect the logic behind vaaac; a single header file library which uses OpenCV to compute the angles at which a user's arm is inclined, and to trigger an event right after a very specific motion has been performed by the user's hand.  
I first wrote the library, and then I wrote [**_cv:go_**](https://github.com/soybin/cvgo); an application which overwrites memory values for the player's view angles in the videogame CS:GO with the values provided by vaaac for the user's arm inclination. It also allows the user to shoot whenever the motion is performed by their hand, as I demonstrate in the following video:
<iframe width="560px" height="315px" src="https://www.youtube.com/embed/YiGEf9hP55E?start=1" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

## understanding the problem
The problem would be something along these lines:  
  
_'Make a computer understand the real world, given a two-dimensional pixel matrix.'_  
  
That problem does certainly look very intimidating. Perhaps because we, as humamns, have more data about our surroundings, such as depth, or perhaps because we don't fully understand how we, as humans, understand the world around us in the first place.  

Making the problem a bit more specific will make it easier to approach:  
  
_'Make a computer identify a person, compute at what angle their arms are inclined, and check if the user has performed a trigger motion with their hands, by quickly moving their index finger up and back down.'_  
  
If you lack basic image processing knowledge, looking at this problem may seem intimidating. But it becomes easier as you learn about the existence of some methods commonly used in this field.  
  
The following is how I personally approached the problem, but keep in mind there's virtually an infinite number of ways of approaching this very same problem.  

## getting the user's skin tone
Finding the user's skin tone is crucial to carrying out the next steps of the process. That's because we'll want to differentiate between the user and everything else in the image.
  
One smart solution to finding the user's skin tone can be to use a Haar Cascade in order to find the face of the user, take a sample of the forehead, and apply a threshold. The reson I didn't end up using this option for vaaac is because Haar cascade files tend to be pretty big, and I wanted to write a library as lightweight as possible.
  
What I did instead is quite simple and intuitive, although it requires a bit of work from the user. I asked the user to fill a portion of the screen with a sample of their arm's skin, and to press any button whenever the requirement was met. The program then takes a copy of the pixel matrix within the screen portion area bounds and computes the average color. As seen in the following gif:  
<img src="../../../../gifs/vaaac01.gif" width="50%"/>

## binarization of the image
After finding the user's color skin, the next step is to binarize the image. That means making every pixel of the image either black (if it's not the user's skin) or white (if it's the user's skin).  
First of all we'll convert the RGB image to HSV, that way we can ignore shadows.  
Then we'll check for every pixel in the image whether that pixel is equal to the average color of the user's skin or not. However, this won't work because of how lighting affects the color of the user's skin. Instead, we have to check for every pixel in the image whether that pixel can be found within a range of values. This can be easily done with the OpenCV function ```cv::inRange(cv::Mat, cv::Scalar(skinTone - lowThreshold), cv::Scalar(skintTone + highThreshold), cv::Mat)```.  

## isolate a component with breadth first search 
Breadth first search (bfs from now on) is a basic graph traversing algorithm (take a look [**_here_**](https://en.wikipedia.org/wiki/Breadth-first_search) in case you don't know much about it).  
A matrix such as the recently computed binary image can be easily interpreted as a graph. Now we can make an assumption, or rather a requirement: the elbow of the user must be fixed nearby the center of the image. This is completely natural, as the user would adopt this position anyways to get more screen space to aim.  
By creating this restriction, we can have a small area in the center of the image that serves as the trigger to identify the bounds of the user's arm.
This means that we can check every pixel of that small area for every frame, and apply bfs only if skin has been found.  
By applying bfs we can get the bounds of the area occupied by the arm, as well as the furthest point from the center of the screen, in other words, the position of the tip of the index finger of the user's arm.  

It's worth noting that applying bfs for every single pixel would be extremely inefficient, so I added a variable _**n**_ that represents the square root of the area in pixels of each new subdivision of the image.  
That just means that I don't apply bfs to every individual pixel, rather I compute the average value for every subdivision of the screen with area \\(2^n\\).  
  
The bfs algorithm works the following way: 
* For every subdivision inside the area at the center of the image, check the average color of the subdivision.
	* If a subdivision has non-zero value, stop checking every subdivision inside the area at the center of the image, and start bfs by adding this subdivision to the bfs queue for processing.
* While the bfs queue is not empty
	* Update \\((x, y)\\) coordinate values to the current subdivision position coordinates (relative to the center of the screen). Since bfs will check the farthest point in the last place, no check is necessary.
	* Check every neighbor subdivision of the currently processed subdivision, and enqueue the neighbors with non-zero value.
  
This is a simplification, but the actual implementation in vaaac is not that different.
```
for (; !q.empty(); ) {
	std::pair<int, int> xy = q.front();
	int x = xy.first, y = xy.second;
	q.pop();
	if (x < 0 || y < 0 || x + BFS_SAMPLE_SIZE > res || y + BFS_SAMPLE_SIZE > res || visited[x][y]) continue;
	visited[x][y] = true;
	if (cv::mean(mask(cv::Rect(x, y, BFS_SAMPLE_SIZE, BFS_SAMPLE_SIZE)))[0] == 0) continue;
	/*
	 * update aiming point coordinates.
	 * works because the bfs algorithm always
	 * visits the farthermost element in the
	 * last place
	 */
	xAim = x;
	yAim = y;
	// update found object area bounds
	xMin = std::min(xMin, x);
	yMin = std::min(yMin, y);
	xMax = std::max(xMax, x);
	yMax = std::max(yMax, y);
	// add neighbors to queue
	for (auto& offset : bfsOffsets) {
		q.push(std::make_pair<int, int>(x + offset.first, y + offset.second));
	}
}
```
  
All that's left to do is compute the aiming angles in degrees relative to the image bounds, and smooth them by an arbitrary factor.
```
// make angles
yAngle = -(double)(halfRes - yAim) / halfRes * 90.0;
xAngle = -(double)(halfRes - xAim) / halfRes * 90.0;
yAngleSmooth += (yAngle - yAngleSmooth) / (double)(AIM_SMOOTHNESS);
xAngleSmooth += (xAngle - xAngleSmooth) / (double)(AIM_SMOOTHNESS);
```

## event trigger
I wanted to have some way for the user to communicate an action to the library, because just computing the arm inclination angles is probably not enough for most usage cases.  
The motion of moving your index finger up and back down would be pretty convenient, since the main idea I had in mind for this library was to integrate it with an fps videogame, so that's what I did.  
  
This problem is fairly easy, but it needed some more work than I initially expected.  
The data structure I used to solve it is a double ended queue, ```std::deque<T>``` in c++ stl. This is because we'll want to remove elements from the back and add to the front, but still be able to access the elements in between.  
Anyways, it boils down to the following:
* If the deque is empty. If that's the case, add the current index finger coordinates to it and continue.
* Else if the bool ```increment``` is true. If that's the case, check if the y-coordinate of the index finger is greater than that of the last deque element. Otherwise, check if the distance between the last enqueued y-coordinate and the first enqueued y-coordinate is greater or equal than a certain threshold (this is to avoid noise from being detected as an event). If that's the case, set ```increment``` to false. Otherwise, clear the deque.
* Else if ```increment``` is set false, check if the y-coordinate of the index finger is smaller than that of the last deque element. Otherwise, check if the distance between the last enqueued y-coordinate and the first enqueued y-coordinate is greater or equal than a certain threshold (this is to avoid noise from being detected as an event). If that's the case, set the boolean ```triggered``` to true meaning that, in this frame, an event was triggered by the user. Then clear the deque and set ```increment``` to true.
  
The following is the code used in vaaac to implement the previous concept. Remember that you can take a full look to the source code at [_**https://github.com/soybin/vaaac**_](https://github.com/soybin/vaaac)
```
if (!yxDeltaSize) {
	yxDelta.push_back({ yAim, xAim });
	yxDeltaSize = 1;
} else if (increment) {
	if (yAim <= yxDelta[yxDeltaSize - 1].first) {
		++yxDeltaSize;
		yxDelta.push_back({ yAim, xAim });
	} else {
		int yDelta = yxDelta[0].first - yxDelta[yxDeltaSize - 1].first;
		if (yDelta >= TRIGGER_MINIMUM_DISTANCE_PIXELS && yDelta <= TRIGGER_MAXIMUM_DISTANCE_PIXELS) {
			increment = false;
		} else {
			yxDeltaSize = 0;
			yxDelta.clear();
		}
	}
} else {
	if (yAim >= yxDelta[yxDeltaSize - 1].first) {
		yxDelta.push_back({ yAim, xAim });
		++yxDeltaSize;
	} else {
		std::pair<int, int> left = yxDelta[0];
		std::pair<int, int> right = yxDelta[yxDeltaSize - 1];
		bool ok = true;
		ok &= std::abs(left.first - right.first) <= TRIGGER_ALLOWED_Y_DEVIATION_PIXELS;
		ok &= std::abs(left.second - right.second) <= TRIGGER_ALLOWED_X_DEVIATION_PIXELS;
		if (ok) {
			triggered = true;
			yAim = left.first;
			xAim = left.second;
		}
		increment = true;
		yxDelta.clear();
	}
}
```

## conclusion
This was a very fun project to work on. I had never done anything related to image processing, but the learning curve was mostly linear thanks to OpenCV's design and comprehensive documentation.  
Computer vision has an exciting long way to go. The possibilities are very exciting and, in the literal sense of the word, endless.  
