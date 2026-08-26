# 3D NUCLEAR ORIENTATION AND NEIGHBOR ALIGNMENT ANALYSIS

## OVERVIEW

This Python pipeline analyzes the orientation of segmented cell nuclei in 3D and quantifies their alignment with a structural tissue axis.

The analysis has two complementary parts:

1. **Nucleus–tissue alignment**

   The 3D shape of each nucleus is analyzed using Principal Component Analysis (PCA). The main nuclear axis is then compared with the local tangent of a tissue structure identified from the X-axis signal.

2. **Neighbor-to-neighbor alignment**

   The 3D orientations of neighboring nuclei are compared using the absolute dot product between their principal axes.

The pipeline produces visualization figures and exports quantitative measurements in CSV and Excel formats.

---

## 1. INPUT DATA

The code requires two 3D TIFF files.

### 1.1 Nuclear segmentation

The first TIFF file must contain a labeled 3D nuclear segmentation.

Each nucleus must have a unique integer label:

- 0 = background
- 1 = nucleus 1
- 2 = nucleus 2
- 3 = nucleus 3
- ...

The code identifies all non-zero labels and analyzes each nucleus independently.

### 1.2 X-axis signal

The second TIFF file contains the 3D signal representing the structural tissue axis.

This signal is used to determine the local geometry of the tissue and calculate a local tangent direction for each nucleus.

---

## 2. REQUIRED PYTHON LIBRARIES

The code uses the following libraries:

```python
import numpy as np
import tifffile
import matplotlib.pyplot as plt
import pandas as pd

from sklearn.decomposition import PCA
from scipy.spatial import KDTree
from scipy.ndimage import binary_fill_holes, binary_erosion
from skimage import measure
from skimage.morphology import disk
```

The main packages are:

- NumPy — numerical calculations and vector operations
- tifffile — reading 3D TIFF images
- Matplotlib — visualization
- Pandas — data handling and CSV/Excel export
- scikit-learn — PCA calculation
- SciPy — KDTree spatial searches and image morphology
- scikit-image — contour extraction and morphological operations

---

## 3. INSTALLATION

Create a Python environment and install the required packages:

```bash
pip install numpy tifffile matplotlib pandas scipy scikit-learn scikit-image openpyxl
```

openpyxl is required by Pandas to export the neighbor-alignment results to Excel.

---

## 4. CONFIGURATION

Before running the script, modify the paths to your dataset.

The main configuration section is located near the beginning of the script.

```python
path = r"/path/to/your/dataset"

output_dir = os.path.join(
    path,
    "figures_output"
)

mask_file = os.path.join(
    path,
    "nuclear_segmentation.tif"
)

axe_x_file = os.path.join(
    path,
    "axeX.tif"
)
```

The script automatically creates the output directory if it does not already exist.

---

## 5. MAIN ANALYSIS PARAMETERS

Several parameters control the image processing and orientation analysis.

### 5.1 Number of histogram bins

```
NB_BINS = 18
```

This defines the number of bins used in the polar histogram of nuclear–tissue angles.

### 5.2 Nuclear orientation correction threshold

```
ANGLE_CORRECTION_THRESHOLD = 10.0
```

If the calculated angle between the nuclear axis and the tissue tangent is less than or equal to 10°, the nuclear axis is reoriented to follow the local tissue tangent.

This is intended to avoid orientation instability when the two vectors are almost parallel.

### 5.3 Base Point

```
DEFAULT_BP_X = 125.0
DEFAULT_BP_Y = 50.0

BASE_POINT = np.array([
    DEFAULT_BP_X,
    DEFAULT_BP_Y
])
```

The Base Point (BP) defines the direction used to orient the local tissue tangent. The tangent is oriented toward this point. The Base Point can also be entered interactively when the script is executed.

### 5.4 Reference Line

```
DEFAULT_RL_AXIS = 2
```

The reference line provides a global polarity cue for orienting the nuclear PCA axis.

The reference line can be defined along either:

- X
- Y

The coordinate of the reference line is entered interactively. For example:

```
X = 200
```

defines a vertical reference line at X = 200.

### 5.5 Contour processing parameters

```
NEW_DECALAGE_PIXEL = 1
SMOOTHING_CONTOUR_WINDOW = 5
THRESHOLD_VALUE = 0.1
MIN_CONTOUR_LENGTH = 50
```

These parameters control extraction of the tissue contour.

- THRESHOLD_VALUE: Defines the intensity threshold used to convert the X-axis projection into a binary mask.
- NEW_DECALAGE_PIXEL: Controls the optional erosion of the binary mask. Increasing this value moves the detected contour inward.
- MIN_CONTOUR_LENGTH: Contours shorter than this value are ignored.
- SMOOTHING_CONTOUR_WINDOW: Defines the moving-average window used to smooth the selected contour.

---

## 6. X-AXIS PROJECTION

The 3D X-axis signal is projected along the Z-axis using a sum projection:

```python
proj_axe_x_z = np.sum(axe_x_vol, axis=0)
```

This creates a 2D representation of the tissue structure.

The nuclear segmentation is projected using maximum intensity:

```python
proj_mask = np.max(mask_vol, axis=0)
```

These projections are used for the subsequent 2D geometric analysis and visualization.

---

## 7. TISSUE CONTOUR EXTRACTION

The projected X-axis signal is thresholded:

```python
binary_image = image_projection > threshold_value
```

The resulting binary image is processed using hole filling:

```python
binary_image = binary_fill_holes(binary_image)
```

An optional erosion is then applied:

```python
selem = disk(shift_pixels)
shifted_binary_image = binary_erosion(
    binary_image,
    selem
)
```

Contours are detected using:

```python
measure.find_contours()
```

If multiple contours are detected, the longest contour above the minimum length threshold is retained. Finally, the contour is smoothed using a circular moving average. The resulting curve is used as the local tissue reference.

---

## 8. NUCLEAR ORIENTATION USING PCA

For each labeled nucleus, the 3D coordinates of all voxels are extracted:

```python
coords_nucleus = np.argwhere(mask_vol == label_id)
```

PCA is then applied to these coordinates.

```python
pca = PCA(n_components=3)
pca.fit(coords)
```

The first principal component is used as the nuclear long axis:

```python
v_pca = pca.components_[0]
```

The nuclear centroid is calculated as:

```python
center = np.mean(coords, axis=0)
```

Nuclei with insufficient voxel numbers or poorly defined PCA results are excluded from the analysis.

---

## 9. PROJECTION OF THE NUCLEAR AXIS INTO 2D

The PCA coordinates are stored in:

```
[Z, Y, X]
```

For comparison with the 2D tissue contour, the nuclear axis is projected into the X-Y plane:

```python
v_pca_2D = v_pca_3D[[2, 1]]
```

Therefore:

3D PCA vector
```
[Z, Y, X]
```

↓

2D vector
```
[X, Y]
```

---

## 10. FINDING THE LOCAL TISSUE REFERENCE

For each nucleus, the nearest point on the smoothed tissue contour is found using a KDTree:

```python
i_pixel_initial = kd_tree_axe_x_lissé.query(center_2D)[1]
```

This provides an initial estimate of the local tissue-axis position. The position is then refined using the X-axis intensity. A local Region of Interest (ROI) is examined around the initial intersection point, and the location with the maximum X-axis intensity is selected. This refined position becomes the final anchor point for calculating the local tissue tangent.

---

## 11. CALCULATION OF THE TISSUE TANGENT

The local tangent is calculated using two points on the smoothed contour. The distance between these points is controlled by:

```
TANGENT_WINDOW_SIZE = 15
```

The tangent is calculated approximately as:

```python
v_tangent = point_next - point_previous
```

and normalized to a unit vector. The tangent is then oriented toward the predefined Base Point. This ensures a consistent direction of the tissue reference across the image.

---

## 12. ORIENTATION OF THE NUCLEAR PCA AXIS

PCA does not provide an intrinsic direction. For example:

```
→→→

and

←←←
```

represent the same principal axis mathematically.

The script therefore uses the Reference Line to establish a consistent polarity. The nuclear axis is oriented according to its position relative to the Reference Line. The default convention in the current code is to orient the nuclear axis away from the Reference Line.

---

## 13. NUCLEAR–TISSUE ANGLE

The final angle between the nuclear long axis and the tissue tangent is calculated using the dot product:

```python
dot_prod = np.dot(
    v_tangent_orienté,
    v_pca_orienté_rl
)

angle = np.degrees(
    np.arccos(
        np.clip(dot_prod, -1.0, 1.0)
    )
)
```

The angle ranges from:

- 0°   = same direction
- 90°  = perpendicular
- 180° = opposite direction

The final angle is stored for every successfully analyzed nucleus.

---

## 14. ORIENTATION CORRECTION

A correction is applied when:

```
angle <= ANGLE_CORRECTION_THRESHOLD
```

With the default settings:

```
ANGLE_CORRECTION_THRESHOLD = 10°
```

When this condition is satisfied, the nuclear axis is reoriented in the same direction as the local tissue tangent. The angle is then recalculated using the corrected nuclear axis. The script records whether the orientation was corrected:

```
'is_corrected': is_corrected
```

---

## 15. GENERATED FIGURES

The script generates a six-panel summary figure.

### Figure 1 — Tissue axis and reference system

Shows:

- X-axis projection
- Smoothed tissue contour
- Base Point
- Reference Line
- Example nucleus

This figure is useful for checking that the global reference system is correctly positioned.

### Figure 2 — Nuclear long axis

Shows the selected nucleus and its final oriented long axis. The figure also displays:

- Nuclear centroid
- Tissue-axis anchor
- Base Point
- Reference Line

This is useful for checking the orientation assigned to an individual nucleus.

### Figure 3 — Nuclear–tissue angle

Provides a zoomed visualization of:

- Nuclear long axis
- Initial tissue tangent
- Oriented tissue tangent
- Nuclear centroid
- Tissue-axis anchor
- Calculated angle

If orientation correction was applied, this is also indicated in the figure.

### Figure 4 — Global nuclear orientation map

All analyzed nuclei are displayed with arrows representing their final nuclear orientation. The arrow color corresponds to the nuclear–tissue angle from 0° → 180°. A colorbar indicates the corresponding angle.

### Figure 5 — Polar histogram

The final nuclear–tissue angles are represented using a polar histogram. The histogram covers 0°–180°. The distribution provides a global overview of nuclear orientation relative to the tissue axis.

### Figure 6 — Summary statistics

The script reports:

- Number of analyzed nuclei
- Number of nuclei excluded
- Number of corrected orientations
- Mean angle
- Standard deviation
- Maximum histogram frequency
- Expected frequency for a uniform distribution

---

## 16. CSV OUTPUT

The per-nucleus results are exported to a CSV file.

The output contains information such as:

| Column                         | Description |
| ------------------------------ | ------------------------------------------ |
| Label_ID                       | Nuclear segmentation ID |
| Center_X                       | X coordinate of nuclear centroid |
| Center_Y                       | Y coordinate of nuclear centroid |
| Reference_Line_Axis            | X or Y |
| Reference_Line_Value           | Position of the reference line |
| Angle_Tangente_Axe_Long_0_180  | Final nuclear–tissue angle |
| Taille_Voxels                  | Number of voxels in the nucleus |
| Axe_Long_Est_Corrige           | Whether orientation correction was applied |

The CSV file is automatically generated at the end of the analysis.

---

## 17. NEIGHBOR ALIGNMENT ANALYSIS

The second part of the script analyzes the spatial alignment of neighboring nuclei.

The maximum neighbor distance is defined as:

```
MAX_DIST_PIXELS = 110
```

Two nuclei are considered neighbors when their 3D centroid distance is smaller than this threshold. The distance is calculated using the 3D nuclear centroids.

---

## 18. 3D NEIGHBOR SEARCH

The nuclear centroids are used to construct a KDTree:

```python
tree = KDTree(coords)
```

For each nucleus, the code searches for other nuclei within 110 pixels. The nucleus itself is excluded from its own neighborhood.

---

## 19. NEIGHBOR ORIENTATION

The principal axis of each nucleus is calculated independently using 3D PCA. Each vector is normalized:

```python
principal_axis = principal_axis / np.linalg.norm(
    principal_axis
)
```

The resulting unit vectors are then compared between neighboring nuclei.

---

## 20. NEIGHBOR ALIGNMENT SCORE

For each neighboring pair, the absolute dot product is calculated:

```python
dot_abs = np.abs(
    np.dot(target_vec, neighbor_vec)
)
```

The absolute value is important because PCA does not define a polarity. Therefore, two nuclei oriented as → and ← are considered to have the same physical axis orientation.

The alignment score ranges from:

- 0 = perpendicular
- 1 = perfectly aligned

---

## 21. NEIGHBOR ALIGNMENT CLASSIFICATION

The script categorizes neighboring pairs according to their alignment score. The current classification implemented in the code is:

- 'Parallèle' if score > 0.7
- 'Moyen' if score > 0.4
- 'Perpendiculaire' otherwise

Therefore:

| Alignment score | Classification |
| ---------------: | -------------- |
| > 0.7            | Parallel       |
| 0.4–0.7          | Intermediate   |
| ≤ 0.4            | Perpendicular  |

Important: this threshold differs from the >0.8 threshold described in the earlier methodological text. The GitHub documentation should follow the actual code unless the threshold is intentional.

---

## 22. EXCEL OUTPUT

All neighboring-cell comparisons are stored in a Pandas DataFrame. The resulting table contains:

- Cellule
- Voisin
- ProduitScalaireAbs

The results are exported as:

```
alignement_voisins.xlsx
```

This file can be opened directly in Excel or imported into statistical software.

---

## 23. EXAMPLE WORKFLOW

A typical analysis proceeds as follows:

1. Prepare 3D nuclear segmentation
2. Prepare 3D X-axis signal
3. Set input/output paths
4. Run the Python script
5. Inspect the X-axis projection
6. Select the nucleus to inspect
7. Define Base Point
8. Define Reference Line
9. Select Z coordinate for visualization
10. Set tangent window
11. Calculate nuclear PCA axes
12. Calculate local tissue tangents
13. Calculate nuclear–tissue angles
14. Generate figures
15. Export nuclear measurements to CSV
16. Calculate neighboring-cell alignment
17. Export neighbor measurements to Excel

---

## 24. RUNNING THE SCRIPT

Run the script from a terminal:

```bash
python nuclear_orientation_analysis.py
```

The program first loads the TIFF files and displays the X-axis projection. The user is then prompted to enter several parameters.

**Step 1 — Select a nucleus for inspection**

The script displays the available nuclear labels. Enter the label ID of the nucleus that should be used for the detailed visualization. If no valid value is entered, the first available label is used.

**Step 2 — Define the Base Point**

Enter:

- Base Point X
- Base Point Y

The Base Point determines the polarity of the tissue tangent.

**Step 3 — Define the Reference Line**

Choose X or Y then enter the coordinate of the reference line.

**Step 4 — Define the Z coordinate**

Select the Z position used for the 3D reference visualization. By default, the middle Z slice is selected.

**Step 5 — Define the tangent window**

Enter the tangent window size. A larger value produces a smoother local tangent but may reduce sensitivity to local curvature.

---

## 25. RECOMMENDED QUALITY CONTROL

Before analyzing a complete dataset, the generated figures should be inspected. In particular, verify:

1. The X-axis contour correctly follows the tissue structure.
2. The Base Point is positioned correctly.
3. The Reference Line provides the intended polarity.
4. The nuclear PCA axes follow the long axis of the nuclei.
5. The tissue tangent follows the local tissue geometry.
6. The angle displayed for individual nuclei is biologically reasonable.
7. The number of nuclei excluded because of PCA failure is low.
8. The neighbor distance produces biologically meaningful neighborhoods.

The six-panel summary figure is particularly useful for identifying problems with the orientation convention.

---

## 26. IMPORTANT PARAMETERS TO ADAPT BETWEEN DATASETS

The following parameters may need to be changed depending on image size, resolution, and tissue morphology:

- THRESHOLD_VALUE
- NEW_DECALAGE_PIXEL
- MIN_CONTOUR_LENGTH
- SMOOTHING_CONTOUR_WINDOW
- TANGENT_WINDOW_SIZE
- TANGENT_ROI_SIZE
- MAX_DIST_PIXELS
- ANGLE_CORRECTION_THRESHOLD

The 110-pixel neighbor distance, in particular, depends on image resolution. It should not automatically be assumed to correspond to the same physical distance in datasets acquired with different resolutions.

---

## 27. OUTPUT FILES

After successful execution, the analysis produces:

```
figures_output/
    Full_Synthesis_ANGLE_TANGENTE_AXE_LONG_...png

Angle_Measures_TANGENTE_AXE_LONG_CORRECTED_....csv

alignement_voisins.xlsx
```

The figures provide visual quality control, while the CSV and Excel files contain quantitative measurements for downstream statistical analysis.

---

## 28. SUMMARY

This pipeline combines 3D nuclear morphology, PCA-based orientation analysis, 2D tissue contour geometry, and 3D spatial neighbor analysis.

The first analysis measures:

How is each nucleus oriented relative to the local tissue axis?

The second analysis measures:

How similarly are neighboring nuclei oriented?

Together, these measurements provide a quantitative description of both global tissue-associated nuclear orientation and local cellular orientational coordination.

---

## CITATION

If you use this code in a publication, please cite the repository and describe the analysis parameters used for your dataset, particularly the contour-processing parameters and orientation-correction settings.
