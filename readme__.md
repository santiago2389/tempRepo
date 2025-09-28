# **3D Volume Pipeline: Cargo Pallet Volume Estimation** 📦

This document provides a technical overview of a 3D volume pipeline designed to accurately estimate the volume of cargo pallets. The system utilizes four asynchronous cameras—left, right, front, and back—to capture multi-view images of a pallet moving through a defined space. The primary challenge addressed is the **scale variance** in the 3D point clouds generated from these asynchronous captures. The pipeline leverages a robust methodology to correct this scaling issue and ensure accurate volume measurement.

---

## **1. Project Overview**

This project solves the problem of accurately measuring the volume of cargo pallets in an automated system. The key technical challenge is the **asynchronous data capture** leading to different scales for each 3D point cloud. Our solution involves a series of steps to first correct the scale and then reconstruct the 3D geometry of the pallet for final volume calculation.

---

## **2. Pipeline Inputs**

The pipeline requires the following inputs for each camera view:

* **Intrinsic Parameters:** Camera calibration matrix ($f_x, f_y, c_x, c_y$).
* **Extrinsic Parameters:** Camera pose (Rotation and Translation relative to a world coordinate system).
* **RGB Image:** The raw color image of the pallet.
* **Pallet Bounding Box:** A 2D bounding box for the pallet in the RGB image.
* **Base Plate ROI (Region of Interest):** A 2D region defining the pallet's base plate.

---

## **3. Pipeline Workflow**

The pipeline is a multi-stage process combining deep learning models with classic 3D reconstruction techniques.

### **3.1. Image Preprocessing & Segmentation**

1.  **Image Undistortion:** Corrects lens distortions using intrinsic parameters to ensure geometric fidelity.
2.  **Depth Estimation:** **Depth Anything V2 Metric small** model generates a **metric depth map** from the undistorted RGB image.
3.  **Pallet Segmentation:** **SAMv2 small/large** model uses the bounding box to create a precise binary mask. This mask is applied to the depth map to get the segmented pallet depth.

### **3.2. Scale Calculation and 3D Reconstruction**

This stage uses the base plate as a fiducial marker to correct for scale variance.

#### **3.2.1. Calculating the Scaling Factor**

1.  **Base Plate Point Cloud Generation:** Pixels within the Base Plate ROI are converted to 3D points $(X_c, Y_c, Z_c)$ using the intrinsic parameters and depth:
    

    $$
    Z_c = \text{depth}(u, v)
    $$

    $$
    X_c = (u - c_x) * Z_c / f_x
    $$

    $$
    Y_c = (v - c_y) * Z_c / f_y
    $$

2.  **3D Dimension Calculation:** An Axis-Aligned Bounding Box (AABB) is computed for the base plate point cloud, yielding its calculated 3D dimensions.

3.  **Deriving the Scaling Factor:** This factor is derived by comparing the known real-world dimensions to the calculated 3D dimensions.

    $$
    \text{Scaling Factor} = \frac{\text{Known Real-World Dimension}}{\text{Calculated 3D Dimension}}
    $$

#### **3.2.2. Scaling the Full Pallet Point Cloud**

1.  **Full Pallet Point Cloud Generation:** A 3D point cloud of the entire pallet is created from the segmented depth map.
2.  **Scaling:** Every point in this cloud is multiplied by the derived **Scaling Factor** to achieve metric accuracy.

### **3.3. Multi-View Integration and Final Volume**

1.  **Coordinate Transformation & Merging:** The scaled point clouds from all four views are transformed into a common world coordinate system using the **extrinsic parameters**, and then merged into a single, comprehensive point cloud.
2.  **Noise Filtering:** Statistical outlier removal algorithms are applied to the merged cloud to eliminate noise and spurious points.
3.  **Volume Estimation:** An AABB is computed for the clean, unified point cloud. The dimensions (length, width, height) of this bounding box are used to calculate the final pallet volume.

---

## **4. Validation and Analysis**

Validation is a critical phase, revealing limitations stemming from the physical setup and current algorithmic implementation.

### **4.1. Systemic and Geometric Challenges**

The primary limitations affecting absolute accuracy include:

* **Significant Perspective Distortion (Non-Uniform Scaling):** The high-angle, **top-down positioning** of the cameras is the chief source of error. This causes objects closer to the camera (the top of the pallet) to be scaled differently than objects further away (the base), introducing a **non-uniform scaling bias** along the vertical (Z) axis. The single scaling factor derived from the base plate *cannot* compensate for this local variation.
* **Residual Lens Distortion:** Despite the undistortion step, **residual radial and tangential distortion** persists. This non-corrected distortion leads to geometric errors, such as *curvature* in what should be flat planes within the point cloud, directly impacting bounding box precision.
* **Inherent Monocular Depth Noise:** The output from the **Depth Anything V2 Metric small** model contains noise, particularly at object edges. This translates to **point cloud "fuzziness,"** which, even after filtering, reduces the precision of the final measured dimensions.

### **4.2. Algorithmic and Implementation Gaps**

The current implementation relies on a basic AABB, which simplifies the volume calculation but compromises accuracy:

* **AABB Limitation vs. Plane Extraction:** The pipeline uses an **Axis-Aligned Bounding Box (AABB)** for volume, which is efficient but less robust. True dimensioning requires the **accurate extraction and fitting of the pallet's major faces (planes)**, a complex task that is **not currently implemented**. The reliance on AABB makes the measurement sensitive to minor rotation and noise.
* **Single Scale Factor Limitation:** The application of a single, uniform scaling factor across the entire point cloud is a practical compromise that assumes a uniform geometric error. This is inadequate for correcting the non-uniform scale variance caused by perspective distortion on taller objects.

### **4.3. Current Measurement Observations**

Validation shows a strong correlation between pallet size and error:

* **Accuracy Bias towards Small/Shallow Pallets:** Pallets with **minimal height** exhibit the highest measurement accuracy because the impact of the perspective error is minimized.
* **Validation with External Tools:** Analysis using external software like **MeshLab** confirms that dimensions for smaller objects align closely with ground truth measurements, while taller objects show greater deviation due to compounded errors.

| Pallet Characteristic | Impact on Accuracy | Root Cause |
| :--- | :--- | :--- |
| **Small/Shallow Height** | **Higher Accuracy** | Perspective error is negligible. |
| **Large/Tall Height** | **Lower Accuracy** | Vertical perspective error severely distorts scale. |
| **Complex Geometry** | **Reduced Robustness** | AABB fails to represent true volume; plane extraction is needed. |



---

## **5. Technical Stack**

| Component | Technology / Library | Role |
| :--- | :--- | :--- |
| **Deep Learning Models** | Depth Anything V2, SAMv2 | Depth Estimation, Semantic Segmentation |
| **3D Processing** | Open3D, PCL | Point cloud generation, transformations, bounding box computation |
| **Core Language** | Python | Main scripting and integration |
| **Inference Engine** | PyTorch, TensorFlow | For running deep learning models |

---

## **6. Pipeline Flowcharts**

### **6.1. Part 1: Scale Estimation Sub-Pipeline**

This part focuses on using the base plate to correct the metric scale of each individual camera's point cloud.

```mermaid
graph LR
    A[Start: Inputs] --> B(Image Undistortion & <br/> RGB/Depth Images);
    B --> C{Base Plate ROI Extraction};
    C --> D[Generate Base Plate <br/> Point Cloud];
    D --> E[Compute 3D Bounding <br/> Box Dimensions];
    E --> F[Calculate Scaling Factor];
    F --> G{Full Pallet Point <br/> Cloud Generation};
    G --> H[Scale Point Cloud <br/> using Factor];
    H --> I[End of Part 1: Scaled Pallet Point Cloud];