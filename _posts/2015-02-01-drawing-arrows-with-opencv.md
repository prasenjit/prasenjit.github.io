---
title: "Drawing Arrows with OpenCV"
date: 2015-02-01 16:38
author: adishavit
categories: [OpenCV, opencv-gems]
tags: [OpenCV, opencv-gems]
---
Here’s a little OpenCV gem I found.  
OpenCV has a (relatively) new function (post-2.4.9) called `cv::arrowedLine()` that let’s you draw a one directional arrow between 2 points.

![img](../../assets/arrow3.png)

It is possible to change the length of the arrowhead tip lines, but not their angle. Strangely, it is not possible to draw a bi-directional arrow.

As of this writing it does not appear in the <a href="http://docs.opencv.org/index.html">official docs</a>, but only in the <a href="http://docs.opencv.org/trunk/modules/imgproc/doc/drawing_functions.html#arrowedline">beta docs here</a>.

```cpp
#include <iostream>
#include <string>

#include <opencv2/opencv.hpp>
#include <highgui.h>

using namespace std;
using namespace cv;

int main(int argc, char* argv[])
{   
   auto width = 320;
   auto height = 320;
   auto img = cv::Mat(cv::Size(width, height), CV_8UC3); // create background image

   auto center = cv::Point(width / 2, height / 2); // center point

   int lineType = 8;
   int thickness = 1;
   double tipLength = 0.1;

   img.setTo(0);                          // clear image - set to black

   for (int angle = 0; angle < 360; angle += 15)
   {
      auto angleRad = angle*CV_PI / 180.0;   // convert angle to radians

      auto length = 150;
      auto direction = cv::Point(length * cos(angleRad), length * sin(angleRad)); // calculate direction
      tipLength = .01 + 0.4 * (angle%180) / 360;

      cv::arrowedLine(img, center +direction*0.5, center + direction, CV_RGB(255, angle, 0), thickness, lineType, 0, tipLength); // draw arrow!

      ++thickness;
      if (0 == angle % 45)
         thickness = 0;

      if (180 <= angle)
         lineType = CV_AA;
   }
   imshow('Arrowed Image', img);          // show image
   waitKey();

   return EXIT_SUCCESS;
}
```

#### References

 - OpenCV 3.0.0 [beta docs](http://docs.opencv.org/trunk/modules/imgproc/doc/drawing_functions.html#arrowedline').


*If you found this post helpful, or you have any questions on this subject, please leave a message in the comments.*
