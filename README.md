
# **Advanced Lane Finding Project**

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
[chessboard]: ./output_images/chessboard.png "Chessboard"
[distortion]: ./output_images/distortion.png "Undistorted"
[threshold]: ./output_images/threshold.png "Thresholded"
[transform]: ./output_images/transform.png "Perspective Transformed"
[margin_search]: ./output_images/margin_search.png "Margin Search"
[window_search]: ./output_images/window_search.png "Windowed Search"
[detection]: ./output_images/detection.png "Detected Lane Lines"


## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points
### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---

### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

The code for this step is contained in the `Camera Calibration` section of the IPython notebook located in `pipeline.ipynb`.  

I start by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image.  `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection. 

I used the `cv2.findChessboardCorners()` and `cv2.drawChessboardCorners()` functions to obtain the result below. I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.

![Chessboard Corners][chessboard]

### Pipeline

#### 1. Provide an example of a distortion-corrected image.

With the camera calibration matrix `mtx` and distortion coefficients `dist`, I applied `cv2.undistort()` to the original image. The code for this can be found in the `Distortion Correction` section.
![Distortion Correction][distortion]

#### 2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

I used a combination of color and gradient thresholds to generate a binary image that combines each thresholding filter into a single image. I first converted the image to from RGB to HLS image components. This allows us to take advantage of the lightness component to deal with shadows in the image. I then performed a white color mask and yellow color mask in order to filter for the white lane lines and yellow lane lines respectively. I applied a Sobel filter to the `l_channel` of the image, thresholding to between (20, 100). 

By stacking the three filters, I produced a combined color visual of what each filter detects. The Sobel filter is shown as red, the yellow mask is shown as green, and the white mask is shown as blue. The three filters were then combined to form the binary image also shown below.

![Thresholded Images][threshold]

#### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

The code for my perspective transform includes a function called `warp()`, which appears in the `Perspective Transform` section. I chose to hardcode the source and destination points by identifying a trapezoid from a straight lane line image.

I found these points to work well:

| Source        | Destination   | 
|:-------------:|:-------------:| 
| 580, 460      | 200, 0        | 
| 270, 670      | 200, 700      |
| 1030, 670     | 1000, 700     |
| 700, 460      | 1000, 0       |

I verified that my perspective transform was working as expected by transforming the straight lane image to a top-down view and finding the lines to be parallel and straight from top to bottom. I also verified with a curved lane image and found the lanes to be parallel as well.

![Perspective Transform][transform]

#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

I started identifying lane line pixels by performing a windowed search across the image with a histogram of the pixels in that window of the image. With a certain number of pixels found, the search would continue in a window based on the position of the pixels found in the last window. All the pixel positions are collected for a second order polynomial fit at the end to determine the lane line. The search windows and subsequent polynomial fit line are shown below.

![Windowed Search][window_search]

After the initial windowed search, I used a margin search based on the polynomial fit of the initial search to find pixels within a certain margin of the polynomial. I found a search margin of 100 pixels to be effective. This margin and the resulting fit line are shown below.

![Margin Search][margin_search]

To smooth out the results, I took an average of the last 10 fitted polynomials to determine a best fit polynomial of the line. This average can be found in the `append_fit()` function of the `Line` class. 

#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

These calculations can be found in the `Accuracy Metrics` section. To calculate radius of curvature, I fit a new second order polynomial to the pixel values converted to real world meter values. This calculation is a function of the first and second derivatives of the fitted polynomial, shown in the `calculate_rad()` function.

The position of the vehicle within the lane is calculated based on the absolute distance of each lane from the vehicle center, in this case simply the center of the input image. The shift of the vehicle from the center of the lane is then simply half the absolute difference in the calculated lane distances. This relies on the assumption that the left and right lanes should be equidistant from the vehicle's center.

#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

The detected lane is displayed onto the original image of the road in the `draw_detection()` function of the section `Overlay on Original Image`. An example is shown below.

![Detection Overlay][detection]

---

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a [link to my video result](https://youtu.be/1f_O_0EsqcQ). Also included is the output video, `output.mp4`.

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

My pipeline has trouble with a large density of shadows over the road, such as in the challenge video. I would further investigate better masking and thresholding techniques that can be applied to the image to filter out or ignore the shadows.

The current implementation also has issues with other non-lane lines that appear on the road. Rather than incorporating these detected lines into the fitted polynomial blindly I could perhaps somehow determine if their x position or radius of curvature makes sense with the x position and radius of curvature of the detected lane up to that point. Lines that are outside of a certain margin of error would be discarded.

