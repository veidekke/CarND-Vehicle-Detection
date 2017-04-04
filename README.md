# P5: Vehicle Detection
### Stephan Brinkmann

The goals / steps of this project are the following:

* Perform a Histogram of Oriented Gradients (HOG) feature extraction on a labeled training set of images and train a classifier Linear SVM classifier
* Optionally, you can also apply a color transform and append binned color features, as well as histograms of color, to your HOG feature vector.
* Note: for those first two steps don't forget to normalize your features and randomize a selection for training and testing.
* Implement a sliding-window technique and use your trained classifier to search for vehicles in images.
* Run your pipeline on a video stream (start with the test_video.mp4 and later implement on full project_video.mp4) and create a heat map of recurring detections frame by frame to reject outliers and follow detected vehicles.
* Estimate a bounding box for vehicles detected.

[//]: # (Image References)
[hog_good]: ./output_images/hog_good.jpg
[hog_bad]: ./output_images/hog_bad.jpg
[windows]: ./output_images/windows.jpg
[heatmaps]: ./output_images/heatmaps.jpg

## [Rubric](https://review.udacity.com/#!/rubrics/513/view) Points
Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---

### Note

I combined the lane finding pipeline from P4 with the vehicle detection. Cells 1-16 contain functions and variables for the lane detection (taken from P4, slightly stripped down, with minor adjustments). The vehicle detection implementation starts in cell 17.

### Writeup / README

#### 1. Provide a Writeup / README that includes all the rubric points and how you addressed each one.  You can submit your writeup as markdown or pdf.  [Here](https://github.com/udacity/CarND-Vehicle-Detection/blob/master/writeup_template.md) is a template writeup for this project you can use as a guide and a starting point.  

You're reading it!

### Histogram of Oriented Gradients (HOG)

#### 1. Explain how (and identify where in your code) you extracted HOG features from the training images.

Cell 17 contains functions for spatial binning, color histogram computation, color conversion, and HOG feature extraction.

Cell 18 contains the `single_img_features(...)` function that computes and returns the desired features for a single image.

The function `extract_features(...)` in cell 19 is basically a wrapper for the former function that works on lists of images (as filenames).

In cell 20, I read in all images from KITTI and GTI sets, split up between vehicles (8792) and non-vehicles (8968).

#### 2. Explain how you settled on your final choice of HOG parameters.

I used the code in cell 22 to experiment with different HOG settings. In a nested for-loop I visualized different combinations of orientation, pixels per cell, and HOG channel on random car and non-car images. Cells per block was always set to `2`, color space was RGB. The visualization is done using the helper function in cell 21.

I then tried to decide which settings led to the most distinctive HOG visualizations.

Here's an example of a pretty bad combination of settings:

![][hog_bad]

These are the settings I finally decided on:
`orientations=9`, `pixels_per_cell=(7, 7)`, `cells_per_block=(2, 2)`, visualized here:

![][hog_good]

Since all 3 channels seem to be pretty significant, I decided to set `hog_channel = 'ALL'`.

(Images visualizing other combinations can be found in the `output_images` directory.)


#### 3. Describe how (and identify where in your code) you trained a classifier using your selected HOG features (and color features if you used them).

I trained a linear SVM using the above-mentioned HOG features with a limited number of samples to try out different values for color features, and ended up with `spatial_size = (32, 32)`, `hist_bins = 32`, and YCrCb as color space because those yielded the highest test accuracy (99.21%). The length of the feature vector was 10,080.

The settings can be found in cell 23, the actual training is done in cell 24.

### Sliding Window Search

#### 1. Describe how (and identify where in your code) you implemented a sliding window search.  How did you decide what scales to search and how much to overlap windows?

The function `find_cars(...)` is defined in cell 25. It uses HOG sub-sampling to extract features from various windows across an image and then predicts their content using the trained SVM. It returns a heatmap with vehicle locations, and optionally draws bounding boxes.

I ran this function on all test images (cell 28), trying different values for `scale`, `y_start`, and `y_stop`, until all cars were recognized and false-positives were mostly avoided.

The final settings I settled for are:

* 2 runs of `find_cars(...)`
* Scale values of 1.3 and 1.5, respectively
* Search areas of 372-592 and 400-656 pixels (y), respectively

This leads to 455 and 392 windows to be searched (847 windows combined), visualized here for both scales:

![][windows]

#### 2. Show some examples of test images to demonstrate how your pipeline is working.  What did you do to optimize the performance of your classifier?

The following image shows the outputs with boxes for both scale values, the combined heatmap, and the resulting bounding boxes (drawn using `draw_labeled_bboxes(...)` from cell 26) after thresholding (using `apply_threshold(...)` from cell 26) and labeling the heatmap. This is shown for all six test images.

![][heatmaps]

To improve performance, I used differently sized search areas for both passes (see above).

A prediction on one frame takes roughly 0.9 seconds.

---

### Video Implementation

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (somewhat wobbly or unstable bounding boxes are ok as long as you are identifying the vehicles most of the time with minimal false positives.)

The video can be found in my submission as `output.mp4`.

#### 2. Describe how (and identify where in your code) you implemented some kind of filter for false positives and some method for combining overlapping bounding boxes.

While cell 29 contains the pipeline for the lane detection from P4, cell 30 is responsible for the vehicle detection. It uses a buffer for the heatmaps, so that they can be summed up over several frames. The resulting summed-up heatmap is then thresholded to reduce false-positives.

Using a buffer length of 20 and a threshold of 30, false-positives are completely eliminated from the project video while cars a smoothly recognized.

The function `process_frame(...)` in cell 31 combines both pipelines. Cell 32 defines and initializes all variables for both pipelines, and loads and processes the video.

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

At first, I only trained on images on the KITTI set to avoid overfitting issues due to the time-series characters of the GTI set. On the plus side, this yielded absolutely no false-positives in the entire video (even without thresholding the heatmap). Unfortunately, the SVM missed too many cars, which meant very wobbly detections.

After retraining the classifier using both KITTI and GTI data, the recognition became more reliable, but false-positives were introduced.

The pipeline might fail when vehicles are in locations where it does not expect them. The windows' positions and sizes are specifically set to work well on the project video, but might have to be expanded to more areas of the frame.

Another issue is the moderate performance of about 1 FPS, which cannot really be considered real-time. To help with processing time:

* the overlap between windows (`cells_per_step`) could be increased.
* the search regions could be cropped on the x-axis as well (leading to a trapezoid-shaped array of windows).
* multi-threading (for example dividing up the two different runs between two threads) could be introduced.
* a more advanced system regarding the windows could be used: only search for vehicles around areas where vehicles have been found in recent frames. To deal with suddenly appearing vehicles, a complete sweep could be done every x frames.

Not only for performance reasons, it might be worth experimenting with a deep-learning approach.
