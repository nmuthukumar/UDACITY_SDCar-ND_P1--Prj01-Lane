# UDACITY -- Self-Driving Car Engineer [NanoDegree] --  
# Part-1--Project-01 - Finding Lane Lines on the Road

## Project Write-up

---

### Goals of the Project
The Goals / Steps of this Project are the following:
- Make a Pipeline that Finds Lane Lines [Left & Right] on the Road from the given Input Images and Videos,  
  and Annotate the detected Lane Lines with:
  - Stage-1: Line Segments
  - Stage-2: Solid Lines (Extrapolated Single Line based on the Line Segments to map out the Full Extent of the Lane Boundaries)
- Reflect on the Project Work in a written Report (This File!).

---

### Reflection

### 1. Description of the Lane Detection Pipeline
My Pipeline consists of the following **6 Steps**:
1. Convert Input Image to Grayscale
2. Smoothing [Gaussian Blur]
3. Edge Detection [Canny Edge Detection]
4. Apply a Region of Interest (ROI) Mask
5. Detect Lines [Hough Lines] & Draw Lines
   - Stage-1: Line Segments
   - Stage-2: Single Solid Line per Lane for its Full Extent.
6. Overlay the Detected Line Segments / Lines on Original Input Image so as to Annotate the Lane Lines

Let us see each Step in detail.

Using the below Example Input Image, I will illustrate the corresponding Output of Each Step.  
![Example Input Image](https://github.com/nmuthukumar/UDACITY_SDCarEngg-ND_P1--Prj01-Lane/blob/master/Pipe/0_SolidWhiteCurve_In.jpg "Image_Input")

#### 1. Convert Input Image to Grayscale  
   Convert the Color Image to a 1-Channel Grayscale Image  
   so that it is convenient to perform the Image Processing Algos in the following Steps.
   
   For this I used the cv2 Function `cv2.cvtColor(img, cv2.COLOR_RGB2GRAY)`,  
   where `img` is the Input Image and  
   `cv2.COLOR_RGB2GRAY` is the Parameter which specifies the Conversion from Color to Grayscale.
   
   This Step's Output for the Example Input Image is:  
   ![Pipeline Step-1: Grayscale Image](https://github.com/nmuthukumar/UDACITY_SDCarEngg-ND_P1--Prj01-Lane/blob/master/Pipe/1_SolidWhiteCurve_O_Gray.jpg "Image_Grayscale")

#### 2. Smoothing [Gaussian Blur]  
   Apply Gaussian Blur so as to Smooth the Image and Prepare for Canny Edge Detection.  
   This is to increase the Accuracy of the Edge Detection by Smoothing Stray Edge Points (Noise).
   
   For this I used the cv2 Function `cv2.GaussianBlur(img, (kernel_size, kernel_size), 0)`,  
   where `img` is the Grayscale Image and  
   `kernel_size` is the Gaussian Kernel Size.  
   I used  
   `kernel_size` = `3`  
   which I find to be an Optimum Value for Smoothing - between Removing Noise and Preserving the Details.
   
   This Step's Output for the Example Input Image is:  
   ![Pipeline Step-2: Smoothened Image](https://github.com/nmuthukumar/UDACITY_SDCarEngg-ND_P1--Prj01-Lane/blob/master/Pipe/2_SolidWhiteCurve_O_Blur.jpg "Image_Blur")

#### 3. Edge Detection [Canny Edge Detection]  
   Apply Canny Edge Detection Algorithm and get the Edges (Prominent Points) in the Image.
   
   For this I used the cv2 Function `cv2.Canny(img, low_threshold, high_threshold)`,  
   where `img` is the Smoothened Image,  
   `low_threshold` is the Lower Threshold for the Hysteresis Procedure and  
   `high_threshold` is the Higher Threshold for the Hysteresis Procedure.  
   I used  
   `low_threshold` = `50` and  
   `high_threshold` = `150`  
   which I find to be Optimum Values for Edge Detection - Both for White and Yellow Lanes.
   
   This Step's Output for the Example Input Image is:  
   ![Pipeline Step-3: Edge Detected Image](https://github.com/nmuthukumar/UDACITY_SDCarEngg-ND_P1--Prj01-Lane/blob/master/Pipe/3_SolidWhiteCurve_O_Edges.jpg "Image_Edges")

#### 4. Apply a Region of Interest (ROI) Mask  
   ROI Mask is applied in order to only Detect the Lane Lines on the Road  
   and to Filter-out the Unwanted Prominent Points by the Road Sides or above the Horizon.
   
   For this, I first Prepared the Mask using the cv2 Function `cv2.fillPoly(mask, vertices, ignore_mask_color)`,  
   where `mask` is a Blank Mask to start with,  
   `vertices` is the Vertices of the Polygon which specifies the Region of Interest and  
   `ignore_mask_color` is the Fill Color for Preserving the Region of Interest while Masking.  
   I formed the `vertices` by using % of Image Size rather than Absolute Points so as to handle Images of Any Size.  
   I used:  
       `0.59` of `y Size` from 0 (Image Height from 0) for fixing Top Vertices of ROI,  
       `0.48` of `x Size` from 0 (Image Length from 0) for fixing Left  Top Vertex of ROI,  
       `0.54` of `x Size` from 0 (Image Length from 0) for fixing Right Top Vertex of ROI, and  
       `Image Bottom Left & Right Corners` (Full Image Height from 0) for fixing Bottom Vertices of ROI  
   as I find these Points to be Optimum for fixing the ROI that includes  
   only Both of the Left & Right Lane Lines up to a Distance unto Horizon.  
   Also, I used `ignore_mask_color` = `255` (White).
   
   I then Applied the Mask to extract the Image only where the Mask Pixels are Non-Zero  
   using the cv2 Function `cv2.bitwise_and(img, mask)` (i.e., Input Image '&' ROI Mask),  
   where `img` is the Edge detected Image and  
   `mask` is the Mask that I prepared above.
   
   This Step's Output for the Example Input Image is:  
   ![Pipeline Step-4: Edge Detected & ROI Masked Image](https://github.com/nmuthukumar/UDACITY_SDCarEngg-ND_P1--Prj01-Lane/blob/master/Pipe/4_SolidWhiteCurve_O_E_ROI.jpg "Image_Edges_ROI")

#### 5. Detect Lines [Hough Lines] & Draw Lines  
   - Stage-1: Detect Line Segments within the ROI from the earlier detected Edges and Draw the Line Segments.
   - Stage-2: Draw a Single Solid Line per Lane, by Filtering, Averaging and Extrapolating the Line Segments, so that we can Annotate its Full Extent.
   
   **For Stage-1 (Detect & Draw Line Segments):**  
   I first Detected the Line Segments  
   using the cv2 Function `cv2.HoughLinesP(img, rho, theta, threshold, np.array([]), minLineLength=min_line_len, maxLineGap=max_line_gap)`,  
   where `img` is the Edge detected & ROI Mask applied Image,  
   `rho` and `theta` respectively are  
   the Distance Resolution in Pixels & Angular Resolution in Radians of the Hough Accumulator,  
   `threshold` is the Hough Accumulator Grid Voting Threshold,  
   i.e., Minimum No. of Points that are required in a Line to deduce a Line,  
   `np.array([])` is an Empty List,  
   `minLineLength` is the Minimum Length of Line Segments and  
   `maxLineGap`is the Maximum allowed Gap between Points on the Same Line to Connect them.  
   I used  
   `rho` = `1` Pixel,  
   `theta` = `1` Degree (π/180 Radians),  
   `threshold` = `32`,  
   `minLineLength` = `6` and  
   `maxLineGap` = `2`  
   which I find to be Optimum Values for Detection of Line Segments in various Usecases.  
   Especially,  
   `minLineLength` = `6` ensures that Small Stray Elements like Lane Paint Distortions, Shadows, etc. are Filtered-out and  
   `maxLineGap` = `2` ensures that only Lane Line Segments in the Same Line are connected and  
   Not the Lane Paint Spills or Patches by the Sides.
   
   I then used the Helper Function `draw_lines(img, lines, color=[255, 0, 0], thickness=4)`  
   in its Simplest Form to simply Draw the Line Segments detected by `cv2.HoughLinesP()`.  
   Here, `img` is a Blank Image to draw the Line Segments and  
   `lines` are the Line Segments detected by `cv2.HoughLinesP()`.  
   This Helper Function in-turn uses the cv2 Function `cv2.line(img, (x1, y1), (x2, y2), color, thickness)` to Draw the Lines,  
   where `(x1, y1)` and `(x2, y2)` are the Co-ordinates of the Line Segments and  
   `color` and `thickness` respectively are the Color and the Thickness of the Lines to be drawn.
   
   This Step's Stage-1's Output for the Example Input Image is:  
   ![Pipeline Step-5 Stage-1: Line Segments](https://github.com/nmuthukumar/UDACITY_SDCarEngg-ND_P1--Prj01-Lane/blob/master/Pipe/5a_SolidWhiteCurve_O_E_LSegs.jpg "Image_LineSegments")
   
   **For Stage-2 (Compute & Draw Solid Lines):**  
   In order to Draw a Single Solid Line per Lane so that we can Annotate its Full Extent,  
   I modified the Helper Function `draw_lines()` to do the following Sequence of Operations on the Line Segments:
   1. Filtering
   2. Averaging
   3. Extrapolation and Drawing of the Solid Lines
   
   Let us see each Operation in detail.
   
   1. *Filtering*  
      1. First I calculated the Center and Slope of each of the Line Segments.
      2. Then I Segregated the Line Segments as belonging to Left or Right Lane based on their Slopes.  
         Considering the Origin (0,0) at the Top Left of the Image,  
         If the Slope of a Line Segment is 0 or Positive, then it belongs to the Lane Line on the Right.  
         Else if the Slope of a Line Segment is Negative, then it belongs to the Lane Line on the Left.  
         I then accordingly Aggregate their Centers and Slopes in separate Lists for Right and Left  
         so that I can later do the Averaging.  
         I also calculate the Line Segments' Lengths and Aggregate in order to do a Weighted Mean based on Lengths.
      3. While doing the Aggregation, in order to Avoid too Horizontal or too Vertical Line Segments,  
         I Filtered-out the Line Segments whose Slopes are Below or Above particular Slope Limits.  
         I did some trials and narrowed down the Slope Limits to:  
         Lower Slope Limit `slope_limit_low` = `0.4` and Higher Slope Limit `slope_limit_high` = `0.9`.  
         I keep particularly the Lower Slope Limit as `0.4`  
         in order to tackle the Challenging Use-Cases in the `"challenge.mp4"` Video and  
         Filter-out the Line Segments caused by the Inclined Road Patches and Tree Shadows.
   
   2. *Averaging*  
      Then, using the Lists created above,  
      I calculated the Weighted (by Length) Average Slope & Average Center of the Line Segments on Each Side [Right, Left].
   
   3. *Extrapolation and Drawing of the Solid Lines*  
      In this last Step, using the Average Slope & Average Center Point of the Right and Left Line Segments that I calculated,  
      I use the following Line Equation and Extrapolate to one Solid Line per Side:  
      `y = Mx` => `(y-y') = M(x-x')`, where `(x,y)` is the Average Center & `M` is the Average Slope.  
      => `x' = x - ((y-y')/M)`, where `(x',y')` -> `(x1',y1')` and `(x2',y2')` are the  
      New Points to be found for Drawing the Solid Lines.
      
      For Right Average Line:  
      Top Point: I find x1 by Substituting y1 as at a Specified Height of img [`0.59` of `y Size` from 0]  
      Bottom Point: I find x2 by Substituting y2 as at the Bottom of img [`100%` of `y Size` from 0]
      
      For Left Average Line:  
      Bottom Point: I find x1 by Substituting y1 as at the Bottom of img [`100%` of `y Size` from 0]  
      Top Point: I find x2 by Substituting y2 as at a Specified Height of img [`0.59` of `y Size` from 0]
      
      Then I Draw the Right Average Line & Left Average Line  
      using the cv2 Function `cv2.line(img, (x1, y1), (x2, y2), color, thickness)` as above.

   This Step's Stage-2's Output for the Example Input Image is:  
   ![Pipeline Step-5 Stage-2: Solid Lines](https://github.com/nmuthukumar/UDACITY_SDCarEngg-ND_P1--Prj01-Lane/blob/master/Pipe/5b_SolidWhiteCurve_O_E_Lines.jpg "Image_Lines")

#### 6. Overlay the Detected Line Segments / Lines on Original Input Image so as to Annotate the Lane Lines  
   Overlay Detected Line Segments / Lines on Original Input Image  
   so that we can see the Annotated Lane Lines on the Original Input Image.
   
   For this I used the cv2 Function `cv2.addWeighted(initial_img, α, img, β, γ)`,  
   where `initial_img` is the Original Input Image,  
   `img` is the Final Output of the Pipeline,  
   `α` is the Weight of `initial_img`, `β` is the Weight of `img` and `γ` is a Scalar added to each Sum.
   
   This Final Step's Output for the Example Input Image is:  
   ![Pipeline Step-6: Line Segments / Lines Overlaid Image](https://github.com/nmuthukumar/UDACITY_SDCarEngg-ND_P1--Prj01-Lane/blob/master/Pipe/6_SolidWhiteCurve_O_Lines.jpg "Image_Output_Lines")

### 2. Some Errors/Issues I faced and the corresponding Handlings I did
1. Python Calculation Errors
   1. For some Particular Frame(s) in the Video `challenge.mp4`,  
      the `Right Line Average Center` _or_ `Left Line Average Center` is somehow NOT a Proper Co-ordinate (List),  
      but it Becomes an Object of Type `numpy.float64` instead of `list`!!!  
      So, Access of Individual `x` or `y` Co-ordinate for the Calculations Causes Error:
      ```
          IndexError: invalid index to scalar variable
      ```
      So, in these Cases, i.e., for those Particular Frames, I Abort and Return Without Drawing the Line!
   
   2. For some Particular Frame(s) in the Video `challenge.mp4`,  
      During Calculation, some Particular `Average Center`'s `x` or `y` Co-ordinate somehow Becomes `NaN`!!!  
      So, Conversion To 'int' so as to Pass the Co-ordinate as Parameter to Plotting Function Causes Error:
      ```
          ValueError: cannot convert float NaN to integer
      ```
      So, in these Cases, i.e., for those Particular Frames, I Abort and Return Without Drawing the Line!

2. Weighted Average Line Issue - Improvement from 'Weighing Slope Only' to 'Weighing BOTH Slope & Center'  
   The Video `challenge.mp4` was really very Challenging in most of the Frames,  
   but for a Particular Frame in the Video I had a very peculiar Issue that  
   the Right Average Line jumped a bit too much to the Left - as shown in the Image below  
   (I was Analysing the Issue by Plotting the Solid Lines on the Edges Image for a better understanding):  
   ![`challenge.mp4` Weighted Average Line Issue](https://github.com/nmuthukumar/UDACITY_SDCarEngg-ND_P1--Prj01-Lane/blob/master/Issues/challenge_Issue-01--1_Line.png "challenge.mp4 Weighted Average Line Issue")
   
   Then I Plotted the Line Segments for that Particular Frame to Analyse further.  
   But somehow I could not figure out what the Issue is from the Plot:  
   ![`challenge.mp4` Weighted Average Line Issue -- Line Segments](https://github.com/nmuthukumar/UDACITY_SDCarEngg-ND_P1--Prj01-Lane/blob/master/Issues/challenge_Issue-01--2_LineSegs.png "challenge.mp4 Weighted Average Line Issue -- Line Segments")
   
   Then, in order to unearth the Cause for the Error,  
   I proceeded to Plot the Filtered Line Segments which contribute to the Solid Line.  
   In order to differentiate the Line Segments into Right & Left for a better understanding,  
   I used Green Color for the Right Line Segments and Sky Blue Color for the Left Line Segments:  
   ![`challenge.mp4` Weighted Average Line Issue -- Filtered Line Segments](https://github.com/nmuthukumar/UDACITY_SDCarEngg-ND_P1--Prj01-Lane/blob/master/Issues/challenge_Issue-01--3_LSFilt+L_m_WA.png "challenge.mp4 Weighted Average Line Issue -- Filtered Line Segments")
   
   Here I could notice that there was a Small Green Line Segment at the Bottom-Centre  
   which must have pulled the Right Solid Line a bit too much to the Left!  
   By now it struck my mind that in the Calculation of the Average Slope & Centre  
   in my Pipeline Step 5. "Detect Lines [Hough Lines] & Draw Lines",  
   I was calculating Weighted Mean only for the Slopes, and for the Centers I was only calculating a Simple Mean!!  
   So, I immediately went ahead and corrected this by calculating Weighted Mean for the Centres too, and here is what I got!!! :-):  
   ![`challenge.mp4` Weighted Average Line Issue -- Corrected Line with m & C Weighted Average](https://github.com/nmuthukumar/UDACITY_SDCarEngg-ND_P1--Prj01-Lane/blob/master/Issues/challenge_Issue-01--4_LSFilt+L_m,C_WA.png "challenge.mp4 Weighted Average Line Issue -- Corrected Line with m,C WA")

### 3. Potential Shortcomings of the Current Pipeline  
Lane Detection Errors due to:
1. Faded Lane Markings, Missing Lane Markings (possibly during Construction)
2. Lighting Conditions
3. Shadows, Road Patches (Construction)
4. Influence of Road Elevations and Curves on Fixed ROI

Use-Cases like these have a great influence and therefore possibly cause Errors in Edge Detection and/or ROI and/or Line Detection.  
So, an Adaptive Pipeline is very essential rather than a Fixed Pipeline!

### 4. Suggestions of Possible Improvements to the Pipeline
1. Color Thresholding based Detection of Lanes - For a Better Detection of Lighter Colored Lanes like Yellow Lanes and Faded Lanes.
2. Improvements in Pipeline to Dynamically Adapt for Poor Lighting Conditions and also for possibly Missing Lane Markings or Distorted Lane Markings due to Construction.
3. Adaptive ROI based on the Environment - Especially to better handle different Road Elevations and Curves.
4. Drawing of Curves instead of Lines so as to be more practically close with the actual Lane Markings on the Road, especially on Road Curves.
5. Drawing Lines by Continuous Averaging of the Past 'n' Frames instead of only from the Current Frame in order to Avoid Sudden Jumps due to Instantaneous Errors (Noise).
6. Usage of PID Algorithm would be an even better idea to get a Continuous Robust Line.

---
