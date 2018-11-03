# **Advanced Lane Finding Project** 

---

**Advanced Lane Finding Project**

The goals / steps of this project are the following:

* Compute the camera calibration matrix and distortion coefficients given a set of chessboard images.
* Apply a distortion correction to raw images.
* Use color transforms, gradients, etc., to create a thresholded binary image.
* Apply a perspective transform to rectify binary image ("birds-eye view").
* Detect lane pixels and fit to find the lane boundary.
* Determine the curvature of the lane and vehicle position with respect to center.
* Warp the detected lane boundaries back onto the original image.
* Output visual display of the lane boundaries and numerical estimation of lane curvature and vehicle position.

[//]: # (Image References)

[image1]: ./example_images/chessboard.png "corners"
[image2]: ./example_images/chessbo.png "distorted"
[image3]: ./test_images/test4.png "original"
[image4]: ./example_images/undistort_test4.png "distorted"
[image5]: ./example_images/binary_test4.png "Warp Example"
[image6]: ./example_images/warped_test4.png "prespctive"
[image7]: ./example_images/polyfit_test4.png "Output"
[image8]: ./example_images/annotated_test4.png "lane drawn"
[video1]: ./out2.mp4 "Video"

## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points

### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---

### Writeup / README

#### 1. Provide a Writeup / README that includes all the rubric points and how you addressed each one.  You can submit your writeup as markdown or pdf.  [Here](https://github.com/udacity/CarND-Advanced-Lane-Lines/blob/master/writeup_template.md) is a template writeup for this project you can use as a guide and a starting point.  

You're reading it!

### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

The code for this step is contained int the function(calibrate_camera())in the first code cell of the IPython notebook located in "./project.ipynb" 

I start by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image.  `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.  

I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  I applied this distortion correction to the test image using the `cv2.undistort()` function and obtained this result: 

![alt text][image1]
![alt text][image2]

### Pipeline (single images)

![alt text][image3]

#### 1. Provide an example of a distortion-corrected image.

To demonstrate this step, I will describe how I apply the distortion correction to one of the test images like this one:
Using the camera calibration matrices from calibration, I undistort the input image. Below is the example image above, undistorted:
![alt text][image4]

#### 2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

The next step is to create a thresholded binary image, taking the undistorted image as input. The goal is to identify pixels that are likely to be part of the lane lines. In particular, I perform the following:

Apply the following filters with thresholding, to create separate "binary images" corresponding to each individual filter
Absolute horizontal Sobel operator on the image
Sobel operator in both horizontal and vertical directions and calculate its magnitude
Sobel operator to calculate the direction of the gradient
Convert the image from RGB space to HLS space, and threshold the S channel
Combine the above binary images to create the final binary image
Here is the example image, transformed into a binary image by combining the above thresholded binary filters:

![alt text][image5]


#### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

Given the thresholded binary image, the next step is to perform a perspective transform. The goal is to transform the image such that we get a "bird's eye view" of the lane, which enables us to fit a curved line to the lane lines (e.g. polynomial fit). Another thing this accomplishes is to "crop" an area of the original image that is most likely to have the lane line pixels.

To accomplish the perspective transform, I use OpenCV's getPerspectiveTransform() and warpPerspective() functions. I hard-code the source and destination points for the perspective transform. The source and destination points were visually determined by manual inspection, although an important enhancement would be to algorithmically determine these points.

Here is the example image, after applying perspective transform:


![alt text][image6]
The code to perform perspective transform is in "./project.ipynb", in particular the function perspective_transform(). For all images in 'test_images/*.jpg', the warped version of that image (i.e. post-perspective-transform) is saved in 'output_images/warped_*.png'.
#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

Given the warped binary image from the previous step, I now fit a 2nd order polynomial to both left and right lane lines. In particular, I perform the following:

Calculate a histogram of the bottom half of the image
Partition the image into 9 horizontal slices
Starting from the bottom slice, enclose a 200 pixel wide window around the left peak and right peak of the histogram (split the histogram in half vertically)
Go up the horizontal window slices to find pixels that are likely to be part of the left and right lanes, recentering the sliding windows opportunistically
Given 2 groups of pixels (left and right lane line candidate pixels), fit a 2nd order polynomial to each group, which represents the estimated left and right lane lines
The code to perform the above is in the line_fit() function in "./project.ipynb".

Since our goal is to find lane lines from a video stream, we can take advantage of the temporal correlation between video frames.

Given the polynomial fit calculated from the previous video frame, one performance enhancement I implemented is to search +/- 100 pixels horizontally from the previously predicted lane lines. Then we simply perform a 2nd order polynomial fit to those pixels found from our quick search. In case we don't find enough pixels, we can return an error (e.g. return None), and the function's caller would ignore the current frame (i.e. keep the lane lines the same) and be sure to perform a full search on the next frame. Overall, this will improve the speed of the lane detector, useful if we were to use this detector in a production self-driving car. The code to perform an abbreviated search is in the tune_fit() function in "./project.ipynb".

Another enhancement to exploit the temporal correlation is to smooth-out the polynomial fit parameters. The benefit to doing so would be to make the detector more robust to noisy input. I used a simple moving average of the polynomial coefficients (3 values per lane line) for the most recent 5 video frames. The code to perform this smoothing is in the function add_fit() of the class Line in "./project.ipynb". The Line class was used as a helper for this smoothing function specifically, and Line instances are global objects in "./project.ipynb".

Below is an illustration of the output of the polynomial fit, for our original example image. For all images in 'test_images/*.jpg', the polynomial-fit-annotated version of that image is saved in 'output_images/polyfit_*.png'.

![alt text][image7]

#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

**Radius of curvature**

Given the polynomial fit for the left and right lane lines, I calculated the radius of curvature for each line according to formulas presented here. I also converted the distance units from pixels to meters, assuming 30 meters per 720 pixels in the vertical direction, and 3.7 meters per 700 pixels in the horizontal direction.

Finally, I averaged the radius of curvature for the left and right lane lines, and reported this value in the final video's annotation.

The code to calculate the radius of curvature is in the function calc_curve() in "./project.ipynb".

**Vehicle offset from lane center**

Given the polynomial fit for the left and right lane lines, I calculated the vehicle's offset from the lane center. The vehicle's offset from the center is annotated in the final video. I made the same assumptions as before when converting from pixels to meters.

To calculate the vehicle's offset from the center of the lane line, I assumed the vehicle's center is the center of the image. I calculated the lane's center as the mean x value of the bottom x value of the left lane line, and bottom x value of the right lane line. The offset is simply the vehicle's center x value (i.e. center x value of the image) minus the lane's center x value.

The code to calculate the vehicle's lane offset is in the function calc_vehicle_offset() in "./project.ipynb".

**Annotate original image with lane area**

Given all the above, we can annotate the original image with the lane area, and information about the lane curvature and vehicle offset. Below are the steps to do so:

Create a blank image, and draw our polyfit lines (estimated left and right lane lines)
Fill the area between the lines (with green color)
Use the inverse warp matrix calculated from the perspective transform, to "unwarp" the above such that it is aligned with the original image's perspective
Overlay the above annotation on the original image
Add text to the original image to display lane curvature and vehicle offset
The code to perform the above is in the function final_viz() in "./project.ipynb".






#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

Below is the final annotated version of our original image. For all images in 'test_images/*.jpg', the final annotated version of that image is saved in 'output_images/annotated_*.png'.

![alt text][image8]

---

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a [link to my video result](./out2.mp4)

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

This is an initial version of advanced computer-vision-based lane finding. There are multiple scenarios where this lane finder would not work. For example, the Udacity challenge video includes roads with cracks which could be mistaken as lane lines (see 'challenge_video.mp4'). Also, it is possible that other vehicles in front would trick the lane finder into thinking it was part of the lane. More work can be done to make the lane detector more robust, e.g. deep-learning-based semantic segmentation to find pixels that are likely to be lane markers (then performing polyfit on only those pixels).
