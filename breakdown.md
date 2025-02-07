### **Simple Summary of the Paper: "Single Image Haze Removal Using Dark Channel Prior"**

#### **Problem:**
Outdoor images often appear hazy due to particles in the air, which reduces visibility and affects colors. The challenge is to remove haze from a single image without needing extra information like depth or multiple images.

#### **Solution:**
The authors propose a **Dark Channel Prior (DCP)**, a simple yet effective method based on a key observation:
- In most natural, haze-free outdoor images, at least one color channel (red, green, or blue) has very low intensity in some areas.
- When an image is hazy, these "dark pixels" become brighter due to light scattering.
- By identifying these pixels, the haze thickness can be estimated and removed.

#### **How It Works:**
1. **Estimate Haze Thickness**: The method detects dark pixels in small patches of the image and uses them to estimate how much haze is present.
2. **Recover Clear Image**: Using a mathematical model, the haze is subtracted from the image, restoring its original contrast and colors.
3. **Estimate Depth Map** (Bonus Feature): Since haze is thicker at farther distances, the method also provides a rough depth map of the scene.

#### **Why This is Good:**
- Works on **single images** (unlike older methods needing multiple images).
- Produces **high-quality, natural-looking results**.
- Can handle **even heavy haze** conditions.

#### **Limitations:**
- May not work well if an object is naturally similar in color to haze (e.g., white buildings).
- The method assumes a uniform haze distribution, which may not always be true.

#### **Conclusion:**
The **Dark Channel Prior** is a powerful, simple, and effective method for removing haze from images. It has applications in photography, computer vision, and even autonomous driving, where clear visibility is crucial.

---

### **Mathematical Breakdown of "Single Image Haze Removal Using Dark Channel Prior"**

The **Dark Channel Prior (DCP)** method for haze removal is based on a **haze imaging model** and a simple mathematical approach to estimate and remove haze from an image. Let’s go step by step.

---

## **1. Haze Imaging Model**
A hazy image is formed due to **atmospheric scattering**, which consists of two components:
1. **Direct Attenuation** – The light coming from an object is reduced by haze.
2. **Airlight** – Light scattered by haze particles blends into the image.

This is modeled by the equation:

$
I(x) = J(x) t(x) + A (1 - t(x))
$

Where:
- \( I(x) \) = Observed hazy image.
- \( J(x) \) = Haze-free image (which we want to recover).
- \( t(x) \) = Transmission map (indicates how much scene light reaches the camera).
- \( A \) = Global atmospheric light (intensity of haze).
- \( x \) = Pixel position in the image.

---

## **2. Transmission Estimation Using Dark Channel Prior**
### **Step 1: Definition of the Dark Channel**
The **Dark Channel Prior** is based on the observation that in most **haze-free outdoor images**, at least one of the **RGB channels** has very low intensity in small patches of the image.

For an image \( J(x) \), the **dark channel** is defined as:

$
J_{\text{dark}}(x) = \min_{c \in \{r,g,b\}} \left( \min_{y \in \Omega(x)} J_c(y) \right)
$

Where:
- \( J_c(y) \) is the intensity of the **red (r), green (g), or blue (b) channel** at pixel \( y \).
- \( \Omega(x) \) is a local patch centered at \( x \).
- The **minimum operator** first finds the darkest pixel in each color channel, then picks the smallest among them.

In a haze-free image, **Jdark(x) is close to zero for most patches except the sky**.

---

### **Step 2: Estimating Transmission Map**
In a hazy image, the **dark channel pixels become brighter** because of haze (airlight). Using the haze model equation and the dark channel prior, we can estimate the transmission \( t(x) \):

$
t(x) = 1 - \min_{c} \left( \min_{y \in \Omega(x)} \frac{I_c(y)}{A_c} \right)
$

Where:
- \( I_c(y) \) = The observed intensity in the haze image.
- \( A_c \) = The atmospheric light intensity for each channel.
- \( t(x) \) = Estimated transmission map (how much light passes through the haze).

To avoid completely removing all haze (which can make the image look unnatural), a small constant \( \omega \) is introduced:

$
t(x) = 1 - \omega \min_{c} \left( \min_{y \in \Omega(x)} \frac{I_c(y)}{A_c} \right)
$

where \( \omega \) is a scaling factor (typically set to 0.95).

---

## **3. Recovering the Haze-Free Image**
Once we estimate the transmission map \( t(x) \), we can recover the original scene radiance \( J(x) \) by inverting the haze equation:

$
J(x) = \frac{I(x) - A}{\max(t(x), t_0)} + A
$

where:
- \( t_0 \) is a lower bound for transmission (e.g., 0.1) to **avoid dividing by very small values** and prevent excessive darkening.

---

## **4. Estimating Atmospheric Light \( A \)**
The **atmospheric light** \( A \) is typically estimated by:
1. Finding the **brightest pixels** in the dark channel (regions most affected by haze).
2. Choosing the pixel with the highest intensity in the original hazy image.

This ensures that the brightest, most haze-covered regions are used to estimate \( A \).

---

## **5. Refining the Transmission Map**
To improve accuracy, the transmission map \( t(x) \) is refined using **soft matting** (a method from image processing) to smooth transitions and remove blocky artifacts.

This is solved as an **optimization problem**:

$
E(t) = t^T L t + \lambda (t - \tilde{t})^T (t - \tilde{t})
$

where:
- \( L \) is a matting Laplacian matrix (ensures smooth transitions).
- \( \lambda \) is a regularization parameter.
- \( \tilde{t} \) is the initial transmission map.
- The goal is to refine \( t(x) \) while keeping it close to the estimated values.

---

## **Summary of Key Equations**
1. **Haze Imaging Model**:  
   $
   I(x) = J(x) t(x) + A (1 - t(x))
   $
2. **Dark Channel Prior**:  
   $
   J_{\text{dark}}(x) = \min_{c} \left( \min_{y \in \Omega(x)} J_c(y) \right)
   $
3. **Transmission Estimation**:  
   $
   t(x) = 1 - \omega \min_{c} \left( \min_{y \in \Omega(x)} \frac{I_c(y)}{A_c} \right)
   $
4. **Haze-Free Image Recovery**:  
   $
   J(x) = \frac{I(x) - A}{\max(t(x), t_0)} + A
   $
5. **Transmission Map Refinement (Optimization)**:  
   $
   E(t) = t^T L t + \lambda (t - \tilde{t})^T (t - \tilde{t})
   $

---

## **Final Thoughts**
- The **Dark Channel Prior** allows **single-image haze removal** using simple math based on pixel intensity statistics.
- The **transmission map** helps estimate **haze thickness** and even provides a **depth map** as a by-product.
- The method **outperforms older approaches** and works well even in heavy haze conditions.
