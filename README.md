Overview

This analysis pipeline quantifies the orientation of cell nuclei relative to a global tissue axis and measures the degree of orientational coordination between neighboring cells.

The workflow starts from 3D volumetric microscopy data containing:

A labeled nuclear segmentation, where each nucleus has a unique integer label.
A second 3D image representing a structural tissue axis, referred to as the X-axis signal.

The analysis combines 3D nuclear geometry with a 2D representation of the tissue structure. For each nucleus, the pipeline determines:

- The position of the nucleus.
- Its principal axis of elongation.
- The local direction of the tissue axis.
- The angle between the nuclear axis and the tissue axis.
- The orientation of neighboring nuclei.
- The degree of alignment between neighboring nuclei.


1. Input Data

The pipeline requires two volumetric images with Nuclear segmentation tha allow to get the fisrt input as a 3D labeled segmentation mask.

Each nucleus is represented by a unique integer value:

0 = background
1 = nucleus 1
2 = nucleus 2
3 = nucleus 3
...

The coordinates of all voxels belonging to each nucleus are extracted from this volume.


The second input is a 3D intensity volume representing the structural axis of the tissue. (also segmented)

This signal is used to establish a geometric reference against which nuclear orientation can be measured.



2. Generation of 2D Tissue Projections

Although the original data are 3D, the tissue axis is analyzed in the 2D imaging plane.

Nuclear projection is obtained thanks to a maximum-intensity projection of the nuclear segmentation 
This provides a 2D visualization of the spatial distribution of the nuclei.

Conceptually:

nuclear_projection = np.max(nuclear_mask, axis=0)

This projection is primarily used for visualization and for displaying nuclear orientation vectors.


The X-axis signal is instead summed along Z:

x_projection = np.sum(x_signal, axis=0)

Using a sum projection preserves information about the total structural signal along the Z direction.

The resulting image represents the tissue axis as a continuous 2D intensity distribution.

3. Detection of the Tissue Axis

The projected X-axis signal is converted into a binary mask.

Thresholding

Pixels above a selected intensity threshold are classified as belonging to the X-axis structure:

X-axis projection
       │
       ▼
   Threshold
       │
       ▼
 Binary mask
Hole filling

Small holes inside the binary structure are filled to produce a more continuous region.

This is important because discontinuities in the binary mask can lead to fragmented or unstable contours.

Optional erosion

The binary mask can optionally be eroded by a small number of pixels.

Erosion moves the detected boundary slightly inward and can reduce irregularities caused by noisy edges.

4. Contour Extraction and Smoothing

The external contours of the binary tissue mask are detected.

If several contours are present, the pipeline retains the longest contour above a predefined minimum length.

This removes small disconnected structures and focuses the analysis on the main tissue axis.

The selected contour is then smoothed using a circular moving-average filter.

The purpose of smoothing is to remove small local fluctuations while preserving the overall geometry of the tissue boundary.

The result is a continuous 2D curve:

Raw contour
     │
     ▼
Moving-average smoothing
     │
     ▼
Smoothed tissue contour

This smoothed contour becomes the geometric reference used to calculate the local tissue direction.

5. Nuclear Principal Axis Using PCA

For every segmented nucleus, all 3D voxel coordinates are extracted.

For example:

(x1, y1, z1)
(x2, y2, z2)
(x3, y3, z3)
...

These coordinates are analyzed using Principal Component Analysis (PCA).

PCA identifies the direction in which the nuclear voxel coordinates have the greatest variance.

The first principal component is therefore used as the nuclear long axis.

Conceptually:

Nuclear voxel coordinates
          │
          ▼
         PCA
          │
          ├── PC1 → nuclear long axis
          ├── PC2 → secondary axis
          └── PC3 → third axis

The first principal component is normalized to a unit vector.

Nuclei with too few voxels or poorly defined PCA results are excluded because their orientation cannot be determined reliably.

6. Nuclear Centroids

The centroid of each nucleus is calculated from its voxel coordinates.

The 3D centroid is then projected into the 2D imaging plane.

This projected centroid is used to associate each nucleus with its nearest location on the tissue contour.

7. Finding the Local Tissue Direction

For each nucleus, the closest point on the smoothed tissue contour is initially identified using a nearest-neighbor search.

However, the geometrically closest point is not necessarily the best representation of the biological tissue axis.

Therefore, the pipeline performs a local refinement.

Within a defined region around the initial contour point, the X-axis projection is searched for the location with the highest signal intensity.

This allows the contour anchor to be adjusted toward the strongest structural signal.

The final point is used as the local reference position for that nucleus.

8. Calculation of the Local Tissue Tangent

Once the local anchor point is determined, the direction of the tissue axis is estimated from neighboring points along the smoothed contour.

A fixed contour window is used:

Contour points:

       previous points
             ↓
------●--●--●--●--●------
          ↑
      anchor point

A vector between points on either side of the anchor is calculated.

This vector approximates the local tangent to the tissue contour.

The tangent is normalized to a unit vector.

Because a vector has two possible directions, the tangent is then oriented consistently toward a predefined base point.

This ensures that all tissue-axis vectors have the same polarity across the image.

9. Orientation of the Nuclear Axis

PCA determines an axis but does not determine which direction along that axis should be considered positive.

For example, PCA may return:

       ←────────→
       nuclear axis

Both directions describe exactly the same PCA axis.

To obtain a consistent direction, the nuclear axis is oriented relative to a user-defined reference line.

The reference line can be positioned along either the X or Y image axis at a specified coordinate.

The nuclear vector is flipped when necessary so that it points toward the selected reference line.

This provides a consistent polarity convention across nuclei and samples.

10. Special Handling of Nearly Parallel Orientations

The angle between the nuclear axis and the local tissue tangent is normally calculated after both vectors have been oriented.

However, when the two vectors are almost parallel, small numerical or contour variations can produce unstable orientation decisions.

The pipeline therefore uses a 10° threshold.

If the angular difference is less than 10°, the nuclear axis orientation is corrected to match the local tissue tangent.

This prevents artificial orientation flips in cases where the nuclear and tissue axes are already nearly parallel.

11. Nuclear–Tissue Alignment Angle

The angle between the nuclear long axis and the local tissue tangent is calculated using their normalized dot product.

For two unit vectors:

n = nuclear axis
t = tissue tangent

the angle is calculated as:

θ = arccos(n · t)

and converted from radians to degrees.

The resulting angle ranges from:

0°   → parallel
90°  → perpendicular
180° → opposite direction

Thus, the angle provides a measure of how each nucleus is oriented relative to the local tissue geometry.

12. Visualization

Several visualization outputs are generated.

Nuclear orientation map

Nuclear centroids are displayed over the X-axis projection.

A vector representing the nuclear long axis is plotted at each nucleus.

The local tissue tangent can also be plotted for comparison.

This provides a spatial representation of how nuclear orientation changes throughout the tissue.

Individual cell visualization

Selected nuclei can be visualized at higher magnification.

These plots show:

Nuclear position
Nuclear long axis
Local tissue tangent
Relative angle between the two vectors

This is useful for visually validating the orientation calculation.

Angular distribution

The measured nuclear–tissue angles are summarized using a polar histogram.

This provides an overview of the global orientation distribution.

Mean and standard deviation are also calculated as summary statistics.

13. Neighboring Cell Alignment

A separate analysis measures whether neighboring nuclei tend to have similar orientations.

Unlike the tissue-axis analysis, this calculation is performed directly in 3D.

The 3D nuclear centroids are used to construct a spatial search structure.

For each nucleus, neighboring nuclei are identified within a distance threshold of:

110 pixels

The 110-pixel threshold was empirically selected by comparing automatically detected neighbors with manually observed neighborhoods in representative 3D images.

The threshold can therefore be modified depending on image resolution and tissue density.

14. Pairwise Nuclear Alignment

For each neighboring pair of nuclei, the 3D PCA orientation is calculated independently.

The two principal-axis vectors are normalized:

|v1| = 1
|v2| = 1

Their alignment is then calculated using the absolute dot product:

alignment = |v1 · v2|

The absolute value is important because PCA axes have no intrinsic polarity.

For example:

→────────→

and

←────────←

represent the same physical orientation.

Therefore, both should be considered aligned.

The resulting alignment score ranges from:

0 = perpendicular
1 = perfectly aligned
15. Classification of Neighboring Cells

Neighboring pairs are classified according to their alignment score.

The current threshold is:

alignment > 0.8

Pairs above this threshold are classified as parallel/aligned.

Pairs below the threshold are classified as non-parallel.

The analysis therefore converts continuous orientation information into a simple categorical measure of local orientational coherence.

16. Output Data

The pipeline exports the measurements required for downstream statistical analysis.

Depending on the implementation, these measurements can include:

Per-nucleus measurements
Nuclear ID
3D centroid
2D centroid
PCA long-axis vector
Local tissue-axis position
Local tissue tangent
Nuclear–tissue angle
Quality-control information
Per-neighbor-pair measurements
Nuclear ID 1
Nuclear ID 2
3D distance
PCA vector of nucleus 1
PCA vector of nucleus 2
Alignment score
Alignment category

These tables can subsequently be analyzed using Python, R, MATLAB, or other statistical software.

17. Software

The analysis is implemented in Python using standard scientific computing and image-processing libraries.

The main dependencies are:

NumPy
SciPy
scikit-image
scikit-learn
Matplotlib

These libraries are used for:

NumPy — numerical operations and array manipulation
SciPy — spatial searches, filtering, and geometric calculations
scikit-image — image processing, segmentation, morphology, and contour detection
scikit-learn — PCA
Matplotlib — visualization and plotting
