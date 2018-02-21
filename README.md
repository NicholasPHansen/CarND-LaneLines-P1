# **Finding Lane Lines on the Road** 

---

**Finding Lane Lines on the Road**

The goals / steps of this project are the following:
* Make a pipeline that finds lane lines on the road
* Reflect on your work in a written report


[//]: # (Image References)
[original]: ./test_images/solidYellowCurve2.jpg "Original"
[gray_scale]: ./test_images_output/gray_scale.jpg "Grayscale"
[gaussian_blur]: ./test_images_output/gaussian_blur.jpg "GaussianBlur"
[canny_edge]: ./test_images_output/canny_edge.jpg "CannyEdge"
[roi_extraction]: ./test_images_output/roi_extraction.jpg "ROIExtraction"
[hough_lines]: ./test_images_output/hough_lines.jpg "HoughLines"
[result_image]: ./test_images_output/solidYellowCurve2.jpg "ResultImage"

---

## Reflection

## 1. Pipeline description

In order to find the lane lines in a image, I have created a pipeline that can be broken down into the following steps:

1. Grayscale conversion
2. Blurring/low-pass filtering
3. Edge detection
4. Region Of Interrest extraction
5. Line finding

A brief description of each step is provided below, I will demonstrate the effect of each step by using the below image as an example.
![alt_text][original]

### 1.1 - Grayscale conversion 

This step will convert and image from a three channel image (r, g, b), to a single channel image, with values in the range `v = [0; 255]`.
For this step, I use the 
The resulting image is shown below.
![alt text][gray_scale]

### 1.2 - Gaussian Blurring

In order to detect edges, I use a canny-edge detector. But a Canny edge detector is a basic high-pass filter, which means that any noise in an image will be detected as an edge.
For this reason, I apply a gaussian blur to the image, with a kernel-size of 3x3, to smoothen the image.
To shown the result of this operation, notice in the image shown below, how the lane reflectors seem to be "less bright".
![alt text][gaussian_blur]

### 1.3 - Edge Detection

In order to detect edges, I use a Canny-edge detection algorithm.
Using the OpenCV built-in `cv2.Canny()` method, which takes two parameters: `low_threshold` and `high_threshold`.
How these thresholds affect the output of the algorithm, can be described by the pseudo-code below:

```python
    if pixel > high_threshold:
        # Pixel is an edge
    elif pixel < low_threshold:
        # Pixel is not and edge
    else:
        # low_threshold < pixel < high_threshold
        # Look at neighbouring pixels
        if neighbour_pixel > high_threshold:
            # Pixel is an edge
        else:
            # Pixel is not an edge
```
From testing I found the following values to be effective in finding the desired edges:

    CANNY_LOW = 50
    CANNY_HIGH = 150

Using these values results in the image shown below:
![alt text][canny_edge]

### 1.4 - Region Of Interrest extraction

To extract the lane lines, I remove everything else in the image but the lane ahead of the car.
This is done by extracting a Region Of Interest (ROI) in the image.
I have defined the ROI by the following parameter:

    VERTICES = [(0, 1), (0.45, 0.55), (0.55, 0.55), (1, 1)]
    
Where the values in the `VERTICES` list, are gains the be applied to the height and width of an image, to get points on the image, irrespective of image size.  
This means I create a vertice from the bottom left corner of the image, to around the center of image, to the bottom right corner of the image.  
I made the following helper function to perform the calculations for me:

```python
    def calculate_vertices(x_dim, y_dim):
        return [(int(x_dim*x_gain), int(y_dim*y_gain)) for (x_gain, y_gain) in VERTICES]
```

This function will create a `list` of `tuple`s of `x` and `y` pairs, which are the dimension multiplied by the respective gains for each point.

For the image in question, which has a size of `(x, y) = (960, 540)`, this will generate the following list

    vertices = [(0, 540), (432, 297), (528, 297), (960, 540)]

The image with the ROI extracted is shown below:
![alt text][roi_extraction]


### 1.5 - Line finding

To find the lines, I extend the `draw_lines()` function, which uses the built-in OpenCV `cv2.HoughLinesP()` function, which returns a list of lines, which is defined by two end points of that line.

I chose the following approach for detecting the lane lines in the output of the `cv2.HoughLinesP()` function:

1. Calculate the slope of all the lines from HoughLinesP output
2. Split lines into either right or left lane (or none) depeding on the slope
3. Average all the right and left lines to find the center and average slope
4. Calculate the line parameters for the left and right lane lines

The slope of each line is calculated as:

```python
    slope = ((y2-y1)/(x2-x1))
```

I then sort the lines according to the slope, where I have chosen the following criteria

```python
    if slope > 0.5: # Right side lane line
        ...
    elif slope < -0.5: # left side lane line
        ...
    else:
        # Line does not belong to either side of the lane markings
        continue
```

Averaging the `x`, `y` and `slope` values for each of the lines in the left and right lane respectively, the equation for a line can be written as:

    y = slope*x + b
    
From this, the bias term `b` can be found by:

    b = y - slope*x

Where the `x` and `y` average values, and the `slope` found before, are inserted.  
With the lines parameterized, it is possible to calculate any point on the line.
Using the `y` values of the ROI as described in previous sections, the corresponding `x` values can be found using the line equation:

    x = (y - b)/slope

Calculating the left and right lane line using the above formulas, yields the following result:
![alt text][hough_lines]

### Final result

Finally, applying the output of the `draw_lines()` method, to the original image, displays the lines on-top of the lane lines nicely.
![alt text][result_image]

See more image examples in the [test_images_output](./test_images_output/) folder.  

I have also applied the pipeline to the supplied example videos, the results are in the [test_videos_output](./test_videos_output/) folder

---

## 2. Potential Shortcommings

One shortcoming of my approach is the very crude sorting of the slopes of the lines.
Although it has proven successful in detecting the lines in the supplied example images, it may not be possible to detect lines that are not directly ahead of the vehicle or have a large curvature (e.g. a sharp turn).

Another possible issue, could be the line detection parameters (canny thresholds etc.), which is currently tuned for brightly lit images, might not find any lines in a darker image (which I suspect is the case of the challenge video).
Currently, the pipeline will crash if either the left or right (or both) lines are not found, as there is no logic handling this case implemented currently, e.g. this is the case when applying the pipeline to the challenge video.


## 3. Suggest possible improvements to your pipeline

I would like to improve the jittery behaviour of the detected lines in the videos.
This could be done by applying a low-pass filter on the line parameters between images. 
This would mean that the line parameters (`slope` and `bias`) cannot jump from one value to another in an instant.  
This approach would makes sense for images close together in time.

Another potential improvement could be to implement adaptive parameters for both the `cv2.Canny()` and the `cv2.HoughLinesP()` methods, such that the pipeline would work in a broader range of weather/lighting conditions (e.g. nighttime driving or entering/exiting shadowy road areas) 