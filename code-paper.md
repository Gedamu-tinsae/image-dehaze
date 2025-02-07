This Python code implements the **Single Image Haze Removal Using Dark Channel Prior** method described in the paper. Let's break it down **step by step** and explain how each part relates to the mathematical model.

---

## **Step 1: Importing Required Libraries**
```python
import PIL.Image as Image
import skimage.io as io
import numpy as np
import time
from gf import guided_filter
from numba import njit
import matplotlib.pyplot as plt
```
- `PIL.Image` → Handles image reading and processing.
- `skimage.io` → Reads and writes images.
- `numpy` → Handles numerical operations.
- `numba.njit` → JIT compilation for faster execution.
- `matplotlib.pyplot` → Used for visualization.
- `gf.guided_filter` → Implements guided filtering for refining transmission maps.

---
## **Step 2: Compute Dark Channel**
```python
@njit
def compute_dark_channel(src, radius):
    rows, cols, channels = src.shape
    dark = np.zeros((rows, cols), dtype=np.float64)

    # Compute per-pixel minimum manually (Numba doesn't support np.min(axis=2))
    tmp = np.zeros((rows, cols), dtype=np.float64)
    for i in range(rows):
        for j in range(cols):
            tmp[i, j] = min(src[i, j, 0], src[i, j, 1], src[i, j, 2])  # Min across RGB channels

    # Apply minimum filter in a window
    for i in range(rows):
        for j in range(cols):
            rmin = max(0, i - radius)
            rmax = min(rows - 1, i + radius)
            cmin = max(0, j - radius)
            cmax = min(cols - 1, j + radius)
            dark[i, j] = np.min(tmp[rmin:rmax + 1, cmin:cmax + 1])

    return dark
```
### **How it relates to the paper**
- **Implements the Dark Channel Prior (DCP) equation**:
$$
  J_{\text{dark}}(x) = \min_{c} \left( \min_{y \in \Omega(x)} J_c(y) \right)
$$
- The function:
  - First computes the **per-pixel minimum** across **RGB channels**.
  - Then applies a **local minimum filter** in a window (`radius` determines window size).
  - Returns the **dark channel image**.

---
## **Step 3: Compute Transmission Map**
```python
@njit
def compute_transmission(src, dark, Alight, omega, radius):
    rows, cols, _ = src.shape
    tran = np.zeros((rows, cols), dtype=np.double)

    for i in range(rows):
        for j in range(cols):
            rmin = max(0, i - radius)
            rmax = min(i + radius, rows - 1)
            cmin = max(0, j - radius)
            cmax = min(j + radius, cols - 1)
            pixel = (src[rmin:rmax + 1, cmin:cmax + 1] / Alight).min()
            tran[i, j] = 1. - omega * pixel

    return tran
```
### **How it relates to the paper**
- Implements the **transmission estimation equation**:
$$
  t(x) = 1 - \omega \min_{c} \left( \min_{y \in \Omega(x)} \frac{I_c(y)}{A_c} \right)
$$
- Steps:
  1. Divides pixel intensities by **atmospheric light \( A \)**.
  2. Takes the **minimum value** within a local patch.
  3. Computes **transmission \( t(x) \)** using **\( 1 - $omega$ $\times$  min value**.

---
## **Step 4: Define the Haze Removal Class**
```python
class HazeRemoval(object):
    def __init__(self, omega=0.95, t0=0.1, radius=7, r=20, eps=0.001):
        pass
```
### **How it relates to the paper**
- Defines hyperparameters:
  - `omega = 0.95` → Strength of haze removal.
  - `t0 = 0.1` → Prevents extreme values in transmission.
  - `radius = 7` → Window size for local processing.

---
## **Step 5: Load Image**
```python
def open_image(self, img_path):
    img = Image.open(img_path)
    self.src = np.array(img).astype(np.double)/255.
    self.rows, self.cols, _ = self.src.shape
    self.dark = np.zeros((self.rows, self.cols), dtype=np.double)
    self.Alight = np.zeros((3), dtype=np.double)
    self.tran = np.zeros((self.rows, self.cols), dtype=np.double)
    self.dst = np.zeros_like(self.src, dtype=np.double)
```
### **How it relates to the paper**
- Reads an image and converts pixel values to **[0,1]** range.
- Initializes arrays for **dark channel**, **airlight**, **transmission map**, and **final dehazed image**.

---
## **Step 6: Compute Dark Channel**
```python
def get_dark_channel(self, radius=7):
    print("Starting to compute dark channel prior...")
    start = time.time()
    self.dark = compute_dark_channel(self.src, radius)  # Call the optimized function
    print("Time:", time.time() - start)
```
### **How it relates to the paper**
- Calls `compute_dark_channel` to get **dark channel image**.

---
## **Step 7: Estimate Atmospheric Light**
```python
def get_air_light(self):
    print("Starting to compute air light prior...")
    start = time.time()
    flat = self.dark.flatten()
    flat.sort()
    num = int(self.rows * self.cols * 0.001)
    threshold = flat[-num]
    tmp = self.src[self.dark >= threshold]
    tmp.sort(axis=0)
    self.Alight = tmp[-num:, :].mean(axis=0)
    print("Time:", time.time() - start)
```
### **How it relates to the paper**
- Implements **atmospheric light estimation**:
  - Picks **top 0.1% brightest** pixels from the **dark channel**.
  - Computes their **mean intensity** as atmospheric light \( A \).

---
## **Step 8: Compute Transmission**
```python
def get_transmission(self, radius=7, omega=0.95):
    print("Starting to compute transmission...")
    start = time.time()
    self.tran = compute_transmission(self.src, self.dark, self.Alight, omega, radius)
    print("Time:", time.time() - start)
```
### **How it relates to the paper**
- Calls `compute_transmission` to compute **haze thickness** \( t(x) \).

---
## **Step 9: Apply Guided Filter (Refinement)**
```python
def guided_filter(self, r=60, eps=0.001):
    print("Starting to compute guided filter transmission...")
    start = time.time()
    self.gtran = guided_filter(self.src, self.tran, r, eps)
    print("Time:", time.time() - start)
```
### **How it relates to the paper**
- Uses **Guided Filtering** to smooth the transmission map.

---
## **Step 10: Recover the Haze-Free Image**
```python
def recover(self, t0=0.1):
    print("Starting recovering...")
    start = time.time()
    self.gtran[self.gtran < t0] = t0
    t = self.gtran.reshape(*self.gtran.shape, 1).repeat(3, axis=2)
    self.dst = (self.src.astype(np.double) - self.Alight) / t + self.Alight
    self.dst *= 255
    self.dst[self.dst > 255] = 255
    self.dst[self.dst < 0] = 0
    self.dst = self.dst.astype(np.uint8)
    print("Time:", time.time() - start)
```
### **How it relates to the paper**
- Implements **haze removal equation**:
 $$
  J(x) = \frac{I(x) - A}{\max(t(x), t_0)} + A
$$

---
## **Final Execution**
```python
if __name__ == '__main__':
    import sys
    hr = HazeRemoval()
    hr.open_image(sys.argv[1])
    hr.get_dark_channel()
    hr.get_air_light()
    hr.get_transmission()
    hr.guided_filter()
    hr.recover()
    hr.show()
```
- Calls **all functions in sequence** to **dehaze an image**.

