# Centroid X,Y Calculation - Complete Explanation
## Sound Object Visualization Tool

---

## Overview

The tool calculates the **geometric centroid** (center point) of drawn shapes using a **uniform resampling** method that eliminates drawing-speed bias.

**Final Output:** Two coordinates in unit space
- `centroid.x` (range: -10 to +10)
- `centroid.y` (range: -10 to +10)

---

## Step-by-Step Process

### STEP 0: Setup - Coordinate System (Lines 351-353)

```javascript
const CANVAS_SIZE = 1000;        // Canvas is 1000×1000 pixels
const UNIT_RANGE = 10;           // Grid goes from -10 to +10 units
const SCALE_FACTOR = 50;         // 1000 / (10 * 2) = 50 pixels per unit
```

**Coordinate System:**
- Canvas pixels: (0, 0) at top-left to (1000, 1000) at bottom-right
- Unit coordinates: (-10, -10) at bottom-left to (+10, +10) at top-right
- Center of canvas: (500, 500) pixels = (0, 0) units

---

### STEP 1: Convert Canvas Pixels to Unit Coordinates (Lines 400-407)

```javascript
function canvasToUnit(x, y) {
    const centerX = CANVAS_SIZE / 2;      // centerX = 500 pixels
    const centerY = CANVAS_SIZE / 2;      // centerY = 500 pixels
    
    return {
        x: (x - centerX) / SCALE_FACTOR,  // Convert X: pixels → units
        y: (centerY - y) / SCALE_FACTOR   // Convert Y: pixels → units (flip axis)
    };
}
```

**Example Conversions:**
```
Canvas (500, 500) → Unit (0, 0)       [Center]
Canvas (750, 250) → Unit (5, 5)       [Top-right quadrant]
Canvas (250, 750) → Unit (-5, -5)     [Bottom-left quadrant]
Canvas (1000, 0)  → Unit (10, 10)     [Top-right corner]
Canvas (0, 1000)  → Unit (-10, -10)   [Bottom-left corner]
```

**Why Y is flipped:**
- Canvas: Y increases downward (0 at top, 1000 at bottom)
- Math grid: Y increases upward (-10 at bottom, +10 at top)
- Formula: `(centerY - y)` flips the Y-axis

---

### STEP 2: Main Centroid Function Entry Point (Lines 753-776)

```javascript
function calculateCentroid(points) {
    // EDGE CASE 1: No points
    if (points.length === 0) {
        return { area: 0, x: 0, y: 0 };
    }
    
    // EDGE CASE 2: Single point
    if (points.length === 1) {
        const unitPoint = canvasToUnit(points[0].x, points[0].y);
        return { area: 0, x: unitPoint.x, y: unitPoint.y };
    }
    
    // STEP 2a: Convert all canvas points to unit coordinates
    const unitPoints = points.map(p => canvasToUnit(p.x, p.y));
    
    // STEP 2b: Check if shape is closed (for area calculation)
    isShapeClosed(unitPoints);
    
    // STEP 2c: Calculate area (all shapes, closed or open)
    const area = calculatePixelBasedArea(unitPoints, brushSize);
    
    // STEP 2d: Calculate geometric centroid ← THIS IS THE KEY STEP
    const centroid = calculateGeometricCentroid(points);
    
    // STEP 2e: Return results
    return { 
        area: area,
        x: centroid.x,    // Centroid X coordinate
        y: centroid.y     // Centroid Y coordinate
    };
}
```

**Input:** Array of canvas points `[{x: 520, y: 480}, {x: 521, y: 481}, ...]`  
**Output:** `{ area: 12.34, x: 2.45, y: -1.23 }`

---

### STEP 3: Geometric Centroid Calculation (Lines 666-750)

This is where the **actual centroid X and Y** are calculated.

#### STEP 3.1: Handle Edge Cases (Lines 667-674)

```javascript
function calculateGeometricCentroid(points) {
    // EDGE CASE 1: No points
    if (points.length === 0) {
        return { x: 0, y: 0 };
    }
    
    // EDGE CASE 2: Single point
    if (points.length === 1) {
        const unitPoint = canvasToUnit(points[0].x, points[0].y);
        return { x: unitPoint.x, y: unitPoint.y };
    }
```

---

#### STEP 3.2: Convert Points and Calculate Path Length (Lines 676-689)

```javascript
    // Convert canvas points to unit coordinates
    const unitPoints = points.map(p => canvasToUnit(p.x, p.y));
    
    // Calculate total path length and individual segment lengths
    let totalLength = 0;
    const segmentLengths = [];
    
    for (let i = 0; i < unitPoints.length - 1; i++) {
        // Distance between consecutive points (Pythagorean theorem)
        const dx = unitPoints[i + 1].x - unitPoints[i].x;
        const dy = unitPoints[i + 1].y - unitPoints[i].y;
        const length = Math.sqrt(dx * dx + dy * dy);
        
        segmentLengths.push(length);  // Store each segment length
        totalLength += length;         // Sum up total path length
    }
```

**Example:**
```
Points in unit coordinates:
  P0: (0, 0)
  P1: (3, 4)    → segment length = √(3² + 4²) = 5.0 units
  P2: (3, 8)    → segment length = √(0² + 4²) = 4.0 units
  
Total path length = 5.0 + 4.0 = 9.0 units
```

---

#### STEP 3.3: Determine Number of Uniform Samples (Lines 691-699)

```javascript
    if (totalLength === 0) {
        // All points are identical
        return { x: unitPoints[0].x, y: unitPoints[0].y };
    }
    
    // Calculate number of samples for resampling
    const numSamples = Math.max(100, Math.floor(totalLength / 0.1));
    const sampleInterval = totalLength / numSamples;
```

**Purpose:** Create enough samples to accurately represent the path.

**Example:**
```
Path length = 9.0 units
numSamples = max(100, floor(9.0 / 0.1)) = max(100, 90) = 100 samples
sampleInterval = 9.0 / 100 = 0.09 units between samples
```

**Why 100 minimum?** Ensures smooth paths even for very short strokes.

---

#### STEP 3.4: Resample Points Uniformly (Lines 701-731)

**This is the critical step that eliminates drawing-speed bias!**

```javascript
    const resampledPoints = [];
    let currentDistance = 0;      // How far along path we've traversed
    let targetDistance = 0;       // Where next sample should be
    let segmentIndex = 0;         // Which segment we're currently in
    let segmentProgress = 0;      // How far through current segment (0 to 1)
    
    // Always include first point
    resampledPoints.push({ x: unitPoints[0].x, y: unitPoints[0].y });
    
    // Generate uniform samples along the path
    for (let sample = 1; sample < numSamples; sample++) {
        targetDistance = sample * sampleInterval;
        
        // Find which segment contains this target distance
        while (segmentIndex < segmentLengths.length && 
               currentDistance + segmentLengths[segmentIndex] < targetDistance) {
            currentDistance += segmentLengths[segmentIndex];
            segmentIndex++;
        }
        
        if (segmentIndex >= segmentLengths.length) break;
        
        // Interpolate position within the segment
        segmentProgress = (targetDistance - currentDistance) / segmentLengths[segmentIndex];
        
        const p1 = unitPoints[segmentIndex];
        const p2 = unitPoints[segmentIndex + 1];
        
        // Linear interpolation to find sample position
        const x = p1.x + (p2.x - p1.x) * segmentProgress;
        const y = p1.y + (p2.y - p1.y) * segmentProgress;
        
        resampledPoints.push({ x, y });
    }
    
    // Always include last point
    const lastPoint = unitPoints[unitPoints.length - 1];
    resampledPoints.push({ x: lastPoint.x, y: lastPoint.y });
```

**Visual Example:**

```
Original drawn path (variable density from drawing speed):
P0 ------ P1 - P2 - P3 - P4 --------- P5
^slow     ^fast area      ^slow

Uniformly resampled path (equal spacing):
S0 - S1 - S2 - S3 - S4 - S5 - S6 - S7 - S8
^equal spacing all along path
```

**Key Concept:** 
- Original points: Density depends on drawing speed
- Resampled points: Equal spacing along path (0.09 units apart in example)
- Result: Every part of path contributes equally to centroid

**Interpolation Formula:**
```
Given segment from P1 to P2, find point at progress t (0 to 1):
  x = P1.x + (P2.x - P1.x) × t
  y = P1.y + (P2.y - P1.y) × t

Example:
  P1 = (2, 3), P2 = (6, 7), t = 0.5 (midpoint)
  x = 2 + (6 - 2) × 0.5 = 2 + 2 = 4
  y = 3 + (7 - 3) × 0.5 = 3 + 2 = 5
  Result: (4, 5) is midpoint between (2,3) and (6,7)
```

---

#### STEP 3.5: Calculate Centroid from Uniform Samples (Lines 737-749)

**This is where the final centroid X and Y are computed!**

```javascript
    // Calculate centroid as simple average of uniformly sampled points
    let sumX = 0;
    let sumY = 0;
    
    for (const point of resampledPoints) {
        sumX += point.x;
        sumY += point.y;
    }
    
    const centroidX = sumX / resampledPoints.length;
    const centroidY = sumY / resampledPoints.length;
    
    return { x: centroidX, y: centroidY };
}
```

**Formula:**
```
centroidX = (sum of all resampled X coordinates) / (number of samples)
centroidY = (sum of all resampled Y coordinates) / (number of samples)
```

**Complete Example:**

```
Resampled points (100 samples):
  S0:  (0.00, 0.00)
  S1:  (0.27, 0.36)
  S2:  (0.54, 0.72)
  ...
  S50: (3.00, 5.50)
  ...
  S99: (6.00, 8.00)

Sum of X: 0.00 + 0.27 + 0.54 + ... + 6.00 = 245.50
Sum of Y: 0.00 + 0.36 + 0.72 + ... + 8.00 = 387.25

centroidX = 245.50 / 100 = 2.455
centroidY = 387.25 / 100 = 3.873

Final centroid: (2.455, 3.873)
```

---

## Complete Flow Diagram

```
USER DRAWS SHAPE ON CANVAS
         ↓
Points captured at variable density based on drawing speed
  Example: [{x: 520, y: 480}, {x: 521, y: 481}, ..., {x: 580, y: 540}]
         ↓
STEP 1: Convert to unit coordinates (canvasToUnit)
  Canvas (520, 480) → Unit (0.4, 0.4)
  Canvas (521, 481) → Unit (0.42, 0.38)
  ...
  Result: [{x: 0.4, y: 0.4}, {x: 0.42, y: 0.38}, ...]
         ↓
STEP 2: Calculate path length
  Segment 1: √((0.42-0.4)² + (0.38-0.4)²) = 0.028 units
  Segment 2: √((0.44-0.42)² + (0.36-0.38)²) = 0.028 units
  ...
  Total length: 9.0 units
         ↓
STEP 3: Determine number of uniform samples
  numSamples = max(100, 9.0 / 0.1) = 100
  sampleInterval = 9.0 / 100 = 0.09 units
         ↓
STEP 4: Resample uniformly along path
  Sample every 0.09 units along path
  Result: 100 evenly-spaced points
         ↓
STEP 5: Calculate average of uniform samples
  centroidX = (sum of 100 X values) / 100
  centroidY = (sum of 100 Y values) / 100
         ↓
FINAL RESULT: { x: 2.455, y: 3.873 }
```

---

## Why This Method Works

### Problem with Naive Averaging

```javascript
// ❌ WRONG: Naive averaging (biased by drawing speed)
function naiveAverage(points) {
    let sumX = 0, sumY = 0;
    for (const p of points) {
        sumX += p.x;
        sumY += p.y;
    }
    return { x: sumX / points.length, y: sumY / points.length };
}
```

**Issue:** If you draw slowly at the top, more points are captured there, so the average pulls toward the top.

**Example:**
```
Top section (drawn slowly):   100 points over 2 units
Bottom section (drawn fast):   10 points over 2 units

Naive average: Heavily weighted toward top (where more points are)
Result: Centroid biased upward ❌
```

### Solution with Uniform Resampling

```javascript
// ✅ CORRECT: Uniform resampling (unbiased)
function uniformResampleAndAverage(points) {
    // 1. Calculate path length
    // 2. Create uniform samples (same spacing all along path)
    // 3. Average the uniform samples
}
```

**Result:**
```
Top section:   50 uniform samples over 2 units
Bottom section: 50 uniform samples over 2 units

Uniform average: Equal contribution from each section
Result: Centroid at true geometric center ✅
```

---

## Key Line References

| Step | Lines | What It Does | Output |
|------|-------|-------------|---------|
| **Constants** | 351-353 | Define coordinate system | CANVAS_SIZE=1000, SCALE_FACTOR=50 |
| **Canvas→Unit** | 400-407 | Convert pixel to unit coords | (500,500)px → (0,0)units |
| **Main Entry** | 753-776 | Entry point for centroid calc | Calls calculateGeometricCentroid |
| **Edge Cases** | 667-674 | Handle 0 or 1 points | Return early if needed |
| **Path Length** | 676-689 | Calculate total path length | totalLength in units |
| **Sample Count** | 696-699 | Determine number of samples | numSamples = 100+ |
| **Resampling** | 701-731 | Create uniform samples | resampledPoints[] |
| **Final Average** | 737-749 | **Calculate centroid X,Y** | **{x: centroidX, y: centroidY}** |

---

## Mathematical Formulas

### Coordinate Conversion
```
Canvas to Unit:
  unitX = (canvasX - 500) / 50
  unitY = (500 - canvasY) / 50

Unit to Canvas:
  canvasX = unitX × 50 + 500
  canvasY = 500 - unitY × 50
```

### Segment Length (Pythagorean Theorem)
```
length = √((x₂ - x₁)² + (y₂ - y₁)²)
```

### Linear Interpolation
```
Given: Point P₁ = (x₁, y₁), Point P₂ = (x₂, y₂), Progress t ∈ [0, 1]
Result: P = (x, y) where:
  x = x₁ + (x₂ - x₁) × t
  y = y₁ + (y₂ - y₁) × t
```

### Centroid Formula
```
centroidX = (Σ xᵢ) / n  where i = 1 to n resampled points
centroidY = (Σ yᵢ) / n  where i = 1 to n resampled points
```

---

## Example Calculation Walkthrough

### Input: Simple 3-Point Path

```
User draws 3 points on canvas:
  Canvas: (500, 500) → (600, 400) → (700, 500)
```

### Step 1: Convert to Unit Coordinates

```
P₀: (500, 500) → canvasToUnit → (0, 0)
P₁: (600, 400) → canvasToUnit → (2, 2)
P₂: (700, 500) → canvasToUnit → (4, 0)
```

### Step 2: Calculate Path Length

```
Segment 0→1: √((2-0)² + (2-0)²) = √8 = 2.828 units
Segment 1→2: √((4-2)² + (0-2)²) = √8 = 2.828 units
Total length: 2.828 + 2.828 = 5.656 units
```

### Step 3: Determine Samples

```
numSamples = max(100, floor(5.656 / 0.1)) = max(100, 56) = 100
sampleInterval = 5.656 / 100 = 0.05656 units
```

### Step 4: Resample (showing first few)

```
Sample 0: (0.0, 0.0)             [start point]
Sample 1: targetDist = 0.05656   → in segment 0→1, progress = 0.020
         → (0 + 2×0.020, 0 + 2×0.020) = (0.04, 0.04)
Sample 2: targetDist = 0.11312   → in segment 0→1, progress = 0.040
         → (0 + 2×0.040, 0 + 2×0.040) = (0.08, 0.08)
...
Sample 50: targetDist = 2.828    → at boundary of segment 0→1
          → (2.0, 2.0)
...
Sample 100: (4.0, 0.0)           [end point]
```

### Step 5: Calculate Centroid

```
Sum of X: 0 + 0.04 + 0.08 + ... + 2.0 + ... + 4.0 = 200.0
Sum of Y: 0 + 0.04 + 0.08 + ... + 2.0 + ... + 0.0 = 100.0

centroidX = 200.0 / 100 = 2.0
centroidY = 100.0 / 100 = 1.0

Final Centroid: (2.0, 1.0)
```

**Verification:** For this V-shaped path, the centroid at (2.0, 1.0) makes sense geometrically!

---

## Summary

**The centroid (X, Y) is calculated as:**

1. Convert all drawn points from canvas pixels to unit coordinates
2. Calculate the total length of the drawn path
3. Create 100+ samples evenly spaced along the path (uniform resampling)
4. Average the X coordinates of all uniform samples → **centroidX**
5. Average the Y coordinates of all uniform samples → **centroidY**

**Key Innovation:** Uniform resampling eliminates drawing-speed bias, ensuring the centroid represents the true geometric center of the path.

**Output Range:**
- X: -10 to +10 (left to right)
- Y: -10 to +10 (bottom to top)
- Precision: 3 decimal places

**Typical values:** Between -8 and +8 for shapes drawn within the visible grid.
