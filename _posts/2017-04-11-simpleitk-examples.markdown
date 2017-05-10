---
layout: post
title:  "A few examples on how to read medical images in SimpleITK with Python"
date:   2017-04-11
categories: itk python
comments: false
---
I came across a couple of posts on Kaggle where people read in medical images. In my opinion, the easiest way to do this is to use [SimpleITK][simpleitk]. SimpleITK is a convenient procedural wrapper around [Insight Segmentation and Registration Toolkit (ITK)][itk]. ITK is an open-source image analysis toolbox for images in 2, 3 or more dimensions. SimpleITk allows you to enjoy many of the advanced functionalities from ITK in python, such as resampling images, filtering and much more.

The documentation can be a bit terse, so I created a [GitHub repository][github-repo] with a couple of common examples to get you started. If you wish to see other examples, please create an issue on GitHub!

#### Get it at [GitHub][github-repo].


[simpleitk]: http://www.simpleitk.org
[itk]: https://itk.org
[github-repo]: https://github.com/jonasteuwen/SimpleITK-examples
