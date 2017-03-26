**Advanced Lane Finding Project**

Project summary:

* Compute the camera calibration matrix and distortion coefficients given a set of chessboard images.
* Apply a distortion correction to raw images.
* Use color transforms, gradients, etc., to create a thresholded binary image.
* Apply a perspective transform to rectify binary image ("birds-eye view").
* Detect lane pixels and fit to find the lane boundary.
* Determine the curvature of the lane and vehicle position with respect to center.
* Warp the detected lane boundaries back onto the original image.
* Output visual display of the lane boundaries and numerical estimation of lane curvature and vehicle position.

[//]: # (Image References)

[image_dist]: ./camera_cal/calibration3.jpg "Distorted"
[image_undist]: ./output_images/undistort_output.jpg "Undistorted"
[image_road_dist]: ./test_images/test3.jpg "Road Transformed"
[image_road_undist]: ./output_images/test3_output.jpg "Road Transformed"
[binary_edge_image]: ./output_images/apply_transforms.jpg "Binary Example"
[road_top_view]: ./output_images/road_top_view.jpg
[top_view_img]: ./output_images/top_view_img.jpg
[image5]: ./output_images/polyfit2.png "Fit Visual"
[image6]: ./output_images/result.jpg "Output"
[video1]: ./project_result.mp4 "Video"

## Code Details

### Camera Calibration

#### 1. The code for this step is contained in the calibrate_camera method of 'project.py' (line 105)  

The idea is to generate two sets of corresponding points for each calibration image.  One on the actual flat undistorted chessboard (same for all calibration images and having z=0) and one on the distorted chessboard that are obtained using `cv2.findChessboardCorners()` method.

The undistorted flat chessboard points are simply generated from a uniform grid using the numpy mgrid function.  

I loop through all of the calibration images and if `cv2.findChessboardCorners` successfully generates the points representing corners of the distorted image, the two sets of points are appended to chessboard_points and image_points arrays respectively.

These arrays are used to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.

 The resulting configuration parameters are used in  `cv2.undistort()` function to obtain the following results:

Distorted:
![alt text][image_dist]

Undistorted:
![alt text][image_undist]

### Pipeline (single images)

#### 1. The resulting undistort procedure is applied to all of the frames of the video in line 201.  In particular the following image:

![alt text][image_road_dist]

generates the following undistorted image:

![alt text][image_road_undist]

#### 2. Sobel transform was used to obtain information about x and y gradients of the image as well as color thresholds to obtain a binary image representing edges that could potentially compose lane lines.  These methods that are implemented in lines 60-102 and called in lines 464 to 496 of `project.py` work by checking if x,y gradients, gradient direction and magnitude fall within specified range.  The ranges can be obtained by some experimentation.  Here's an example of the output for this step.

![alt text][binary_edge_image]

#### 3. The function called change_view() takes and image as well as camera calibration properties from calibrate_camera() function and returns the top down view of the lane.

To accomplish this `cv2.getPerspectiveTransform()` function is used, which requires 4 points in the original image and the corresponding 4 points in the transformed image in order to calculate the transform matrix.  The user may specify calibrate flag from runtime by calling

`python project.py --calibrate True`

In this case rather than hardcode the four points matplotlib `onclick()` function is used to collect the four points in the original image. The user is asked to select four points in the counter-clockwise manner starting from bottom left.  This is done in lines 173 to 198.

The images below show that the perspective transform is working as expected by drawing the `src` and `dst` points onto a test image and its warped counterpart.
Here are the two images side by side

Original Image

![alt text][image_road_dist]

Top View

![alt text][top_view_img]

Top View (binary thresholded)
![alt text][road_top_view]

<!-- ![alt text][image4] -->

#### 4. The function called detect_lanes() detects the lanes and fills in the corresponding information in left_lane and right_lane Line objects that are passed into it.

The function first checks if lane lines were detected in the previous frame.  If yes, it detects 'hot' pixels in a certain margin around the previously detected and fitted lane curves (lines 231-247).  If no, then it uses the sliding window approach to find the x value with largest concentration of pixels and builds a window around that value. The process is repeated as the window moves up (lines 248-311).

We now have a list of indices corresponding to 'hot' pixels in the region of interest.  The corresponding x and y values are collected in lines 313-323 and fitted with a quadratic polynomial in lines 332 and 342 for left and right lanes respectively. Flags corresponding to whether left and right lanes were detected are set so they are available in the next frame. The quadratic polynomial coefficients are averaged over the last 10 frames for a smoother detection.

There is still an issue of x and y values being in pixels.  Rather than transferring the data points to meters and doing the fitting as suggested in the lecture notes, I used a simple change of variables modifications to the curvature formula to calculate curvature and center offset based on the curves generated in pixels coordinates.  This is done in lines 394-405.  Finally sanity checks are done to decide whether to include this frame as an influence factor in future frames.

Example of quadratic fit to 'hot' pixels

![alt text][image5]

#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

In lines 394-405 the radius of curvature of the lanes as well as the position of the vehicle within the lane is calculated. The points are assumed to be represented by the equation x = a f(yb).  Then dx/dy = a bf'(yb) and d^2x/dy^2 = a b^2 f''(yb), so the values for a and b corresponding to pixels to meter change of variables are used when calculating the radius of curvature.

#### 6. The results are plotted back onto the road.  This is implemented in lines 521-545 in the function `process_image()`.  Here is an example of the result on a test image:

![alt text][image6]

---

### Pipeline (video)

#### 1. The final pipeline used on a test video:[link to my video result](./project_result.mp4)

---

### Discussion

One of the difficulties was choosing appropriate parameters for edge and color thresholding.  I don't think that my final values are optimal so one improvement would be to tweak these parameters to handle more difficult road conditions.  

The main place where the pipeline could fail is if the lanes are not recognized correctly and certain outlier points are considered part of the data to be fitted.  Since a quadratic polynomial is used for fitting, even some outliers could change the curvature of the fitted curve considerably.  To avoid this issue I put in several sanity checks (lines 407-426).  If the checks fail I use my project 1 pipeline on the frame (this is only needed for the challenge video, since on the regular video there is no issue). My pipeline from project 1 does a better job at recognizing the correct lanes since it uses a more advanced confidence value system, but its shortcoming is that it only fits lines as opposed to more general curves.  One improvement for this project would be to make better fusion of the two pipelines and to use straight lane detection from project 1 to help in deciding which points to use for quadratic polynomial fit. I could also use more general curves and splines to get more realistic fits. I plan to pursue this further.
