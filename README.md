## Advanced Lane Finding
[![Udacity - Self-Driving Car NanoDegree](https://s3.amazonaws.com/udacity-sdc/github/shield-carnd.svg)](http://www.udacity.com/drive)

---
The goals / steps of this project are the following:

1. Compute the camera calibration matrix and distortion coefficients given a set of chessboard images. (`1_calibration_camera_chessboard.jpg`)
2. Apply a distortion correction to raw images. (`2_undist.jpg`)
3. Use color transforms, gradients, etc., to create a thresholded binary image. (`3_masking.jpg`)
4. Apply a perspective transform to rectify binary image ("birds-eye view"). (`4_warp.jpg`)
5. Detect lane pixels and fit to find the lane boundary. (`5_lane_pixel_fit.jpg`)
6. Determine the curvature of the lane and vehicle position with respect to center. (`6_curvature.jpg`)
7. Warp the detected lane boundaries back onto the original image. (`7_warp_back.jpg`)
8. Output visual display of the lane boundaries and numerical estimation of lane curvature and vehicle position.

Each step has its corresponding output image (`*.jpg`) stored in directory `output_images/`

---
### [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points
#### 1. Camera calibration
The images for calculating the distortion and 3-D to 2-D mapping matrix are stored in `./camera_cal/calibration*.jpg`.
Firstly, I used `cv2.findChessboardCorners` to find out all those corner points (`corners`) in the images.
Then I used `cv2.calibrateCamera` to calculate the distortion (`dist`) and mapping matrix (`mtx`) given the `corners` pts and their corresponding predifined 3-D pts `objp`

<img src="./output_images/1_calibration_camera_chessboard.jpg" alt="chessboard" height="100%" width="100%">

#### 2. Provide an example of a distortion-corrected image
Here is an example of distortion-corrected image:

<img src="./output_images/2_undist.jpg" alt="camera calibration" height="100%" width="100%">

#### 3. Create a thresholded binary image and provide example
I used magnitude of gradients, direction of gradients, and L and S in HLS color space.
A combined rule is used:

```python
combined[((mag_binary == 1) & (dir_binary == 1)) |\
         ((hls_binary == 1) & (dir_binary == 1) & (bright_binary == 1))] = 1
```

Example masking image is showed:

<img src="./output_images/3_masking.jpg" alt="binary masking" height="100%" width="100%">

Moreover, I used widgets to help tunning the parameters of those masking functions. It can provide instantaneous binary result that really help for accelarating this step. The widgets codes are list here:

```python
def interactive_mask(ksize, mag_low, mag_high, dir_low, dir_high, hls_low, hls_high, bright_low, bright_high):
    combined = combined_binary_mask(image,ksize, mag_low, mag_high, dir_low, dir_high,\
                                    hls_low, hls_high, bright_low, bright_high)
    plt.figure(figsize=(10,10))
    plt.imshow(combined,cmap='gray')
    
interact(interactive_mask, ksize=(1,31,2), mag_low=(0,255), mag_high=(0,255),\
         dir_low=(0, np.pi/2), dir_high=(0, np.pi/2), hls_low=(0,255),\
         hls_high=(0,255), bright_low=(0,255), bright_high=(0,255))
```

<img src="./widgets.png" alt="widgets" height="50%" width="50%">

#### 4. Perspective transform
First, I defined the source and destination of perspective points as follows:

| Source        | Destination   | 
|:-------------:|:-------------:| 
| 585, 460      | 320, 0        | 
| 203, 720      | 320, 720      |
| 1127, 720     | 960, 720      |
| 695, 460      | 960, 0        |

Then the `perspective_warper` function is defined which returns perspective image and the matrix `warpM` as well.
`warM` is needed for the later step which does the *inverse perspective* back to the original image.

```python
perspective_img, warpM = perspective_warper(undist,src,dst)
```

An example is showed here:

<img src="./output_images/4_warp.jpg" alt="perspective" height="100%" width="100%">

#### 5. Lane line pixel and polynomial fitting

I applied a windowing approach to identify the lane pixels

<img src="./lane_pixel_windows.png" alt="camera calibration" height="100%" width="100%">

In this example, I used 9 windows for both lane lines. The window is processed in an order from the buttom to the top.

Pixels are detected by the following function

```python
def identify_lane_pixel(img, lcenter_in, rcenter_in, win_num=9, win_half_width=150, start_from_button=False):
```

* `lcenter_in` and `rcenter_in`are the center of searching windows (x-coordinate) at the bottom of the image.

* `win_num` defines how much window are going to used.

* `win_half_width` refers to the half length of window width

* `start_from_button` indicates how the initial centers of windows are set. Specifically, Let the current window as j and current frame index as i. If `start_from_button=True`, the center of window j will be initally set as window **j-1**. Otherwise, it will be initally set as window **j** in frame **i-1**. Then, by using the initial position just set, the lane pixels are identified if the histogram of that window is high enough. Finally, based on those identified pixels, update the center position of current widnow j.

Next, a simple second order polynomial fitting is applied to both identified pixels

```python
# Fit a second order polynomial to each
left_fit = np.polyfit(lpixely, lpixelx, 2)
right_fit = np.polyfit(rpixely, rpixelx, 2)
```

<img src="./output_images/5_lane_pixel_fit.jpg" alt="lane pixel and fit" height="100%" width="100%">

#### 6. Radius of curvature of the lane and the position of the vehicle

The curvature is calculated based on the following formula. Udacity provides a very good tutorial [here](http://www.intmath.com/applications-differentiation/8-radius-curvature.php) !

```python
a1, b1, c1 = left_fit_coefficients
a2, b2, c2 = right_fit_coefficients
r1 = ((1+(2*a1*height*ym_per_pix+b1)**2)**1.5)/(2*np.abs(a1))
r2 = ((1+(2*a2*height*ym_per_pix+b2)**2)**1.5)/(2*np.abs(a2))
```

> There's no need to worry about absolute accuracy in this case, but your results should be "order of magnitude" correct.

So I divide my result by 10 to make it seems more reasonable. And of course, the "order of magnitude" remains intact.

<img src="./output_images/6_curvature.jpg" alt="lane pixel and fit" height="100%" width="100%">

#### 7. Warp the detected lane boundaries back onto the original image

In order to warp back onto the original image, we need to calculate the inverse of perspective transform matrix `warpM`
just apply `Minv = inv(warpM)` which is `from numpy.linalg import inv`

Then, simply apply `cv2.warpPerspective` with `Minv` as input.

Note: use `cv2.putText` to print the curvature and position onto images

<img src="./output_images/7_warp_back.jpg" alt="lane pixel and fit" height="100%" width="100%">

#### 8. Video

Please see the `project_video_out.mp4` in this repository.

### Discussion

to be continue ...
