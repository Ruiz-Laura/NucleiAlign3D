The overall goal is to measure how the orientation of each nucleus relates to the local orientation of the surrounding tissue, and then to determine how similarly neighboring nuclei are oriented.

The analysis has two main parts:

1. Nuclear orientation relative to the tissue axis

The code starts with:

A 3D labeled nuclear mask, where every nucleus has its own integer ID.
A 3D X-axis signal, representing the structural axis of the tissue.

It then:

       a.Projects the 3D data into 2D
The nuclear mask is maximum-intensity projected along Z for visualization.
The X-axis signal is summed along Z to create a 2D representation of the tissue structure.

       b.Extracts the tissue boundary
The X-axis projection is thresholded to create a binary image.
Optional erosion removes small irregularities around the boundary.
Contours are detected.
The longest sufficiently large contour is selected as the tissue boundary.
A circular moving-average filter smooths the contour.

       c.Calculates the orientation of every nucleus
The 3D coordinates belonging to each nucleus are extracted.
PCA is performed on those coordinates.
The first principal component is taken as the long axis of the nucleus.
This gives a 3D orientation vector.
The nucleus centroid is also calculated.

       d.Finds the local tissue direction
For every nucleus, the closest point on the tissue contour is found.
This point is refined using the X-axis signal.
A tangent vector is calculated from neighboring points on the contour.
This tangent represents the local tissue orientation at that nucleus.

       e.Standardizes vector direction
PCA gives an axis but not a consistent direction: a vector pointing left is mathematically equivalent to one pointing right.
The code therefore uses a predefined reference line to give all nuclear vectors a consistent polarity.
The contour tangent is similarly oriented toward a predefined base point.
If the nuclear axis is almost parallel to the tangent (<10°), the nuclear orientation can be corrected to avoid unstable direction assignments.

       f.Calculates the nuclear/tissue angle
The nuclear orientation and tissue tangent are compared using the dot product.
The resulting angle is expressed between 0° and 180°.
This provides a measure of how well each nucleus follows the local tissue geometry.

For example:

0° → nucleus points in the same direction as the tissue axis.
90° → nucleus is perpendicular to the tissue axis.
180° → nucleus points in the opposite direction.

Depending on whether you care about axis alignment rather than polarity, you may eventually want to treat 0° and 180° as equivalent.


2. Orientation of neighboring nuclei

The second analysis asks a different question:

Do nearby nuclei tend to have similar orientations?

For this:

a. The centroid of every nucleus is calculated in 3D.
b. A spatial nearest-neighbor search is performed.
c. Nuclei within 110 pixels of each other are considered neighbors.
d. PCA is independently calculated for each nucleus.
e. The nuclear orientation vectors are normalized.
f. The absolute dot product between two neighboring nuclear axes is calculated.

The alignment score is:

|v₁ · v₂|

where v₁ and v₂ are the two nuclear orientation vectors.

This produces a value from 0 to 1:

0 → perpendicular nuclei
1 → perfectly parallel nuclei
0.8–1 → classified as parallel
<0.8 → classified as non-parallel

The code then counts the number/proportion of neighboring pairs in each category and exports the pairwise measurements for further statistical analysis.
