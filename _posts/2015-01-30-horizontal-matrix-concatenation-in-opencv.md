---
title: "Horizontal Matrix Concatenation in OpenCV"
date: 2015-01-30 12:27
categories: [OpenCV, opencv-gems, undocumented]
tags: [OpenCV, opencv-gems, undocumented]
---
Here's a little undocumented function to horizontally concatenate two `cv::Mat` matrices (with the same number of rows).

![img](../../assets/o_rat.jpg)

It is used like this:

```cpp
 cv::Mat m1(10, 5, CV_32FC3);
 cv::Mat m2(10, 3, CV_32FC3);
 cv::Mat m3;
 cv::hconcat(m1, m2, m3);
 assert(10 == m3.rows && 8 == m3.cols);

 cv::hconcat(std::vector<cv::Mat>{m1, m2}, m3);
 assert(10 == m3.rows && 8 == m3.cols);

 cv::Mat matArray[] = { m1, m2 };
 cv::hconcat(matArray, 2, m3); // Accept array + size
 assert(10 == m3.rows && 8 == m3.cols);
```

The function `cv::hconcat()` has 2 additional overloads: One for accepting an array of `cv::Mat`s and array size, and another for accepting a `cv::InputArrayOfArrays` which is essentially an `std::vector<cv::Mat>`.

Note that this function will copy all the elements of input matrices and (possibly) reallocate the output matrix.

#### References:

 - Header declaration in OpenCV: [core.hpp](https://github.com/Itseez/opencv/blob/master/modules/core/include/opencv2/core.hpp#L968);
 - [Hat tip](<http://stackoverflow.com/a/9543212/135862>) to SO answer.
 - The image [from here](<http://astro.uchicago.edu/cosmus/projects/marsstereo/>) shows a section of rock that has being ground away by the Rock Abrasion Tool (RAT) of the Opportunity rover on Mars.


_If you found this post helpful, or you have more thought on this subject, please leave a message in the comments._
