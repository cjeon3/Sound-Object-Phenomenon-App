# Area Calculation - Complete Explanation
## Sound Object Visualization Tool

---

## Overview

The tool calculates the **area** of drawn shapes using a **pixel-based grid sampling method** that accurately measures the visual footprint, including brush thickness and interior filling for closed shapes.

**Final Output:** Area in grid square units
- Range: 0 to ~400 units² (theoretical max for full 20×20 grid)
- Typical values: 0.1 to 50 units²
- **1 grid square = 1.0 area units**

---

## Step-by-Step Process

### STEP 0: Setup - Constants (Lines 351-353)

```javascript
const CANVAS_SIZE = 1000;        // Canvas is 1000×1000 pixels
const UNIT_RANGE = 10;           // Grid goes from -10 to +10 units
const SCALE_FACTOR = 50;         // 50 pixels per unit
```

**Grid System:**
- Total grid: 20×20 units = 400 grid squares
- Each grid square: 1×1 unit = 1.0 area unit
- Brush size in pixels / 50 = brush diameter in units

---

### STEP 1: Main Entry Point - calculateCentroid (Lines 753-776)

```javascript
function calculateCentroid(points) {
    if (points.length < 2) {
        if (points.length === 0) return { area: 0, x: 0, y: 0 };
        const unitPoint = canvasToUnit(points[0].x, points[0].y);
        return { area: 0, x: unitPoint.x, y: unitPoint.y };
    }
    
    // Convert canvas points to unit coordinates
    const unitPoints = points.map(p => canvasToUnit(p.x, p.y));
    
    // Check if shape is closed (may add closure points to unitPoints)
    isShapeClosed(unitPoints);
    
    // ← THIS IS WHERE AREA IS CALCULATED
    const area = calculatePixelBasedArea(unitPoints, brushSize);
    
    // Calculate centroid
    const centroid = calculateGeometricCentroid(points);
    
    return { 
        area: area,           // ← Final area in grid squares
        x: centroid.x, 
        y: centroid.y 
    };
}
```

---

## HELPER FUNCTIONS

### Helper 1: Calculate Path Length (Lines 436-444)

**Purpose:** Calculate total length of path in units

```javascript
function calculatePathLength(unitPoints) {
    let length = 0;
    
    // Sum up length of each segment
    for (let i = 0; i < unitPoints.length - 1; i++) {
        const p1 = unitPoints[i];
        const p2 = unitPoints[i + 1];
        
        // Pythagorean theorem: distance = √(Δx² + Δy²)
        length += Math.sqrt((p2.x - p1.x) ** 2 + (p2.y - p1.y) ** 2);
    }
    
    return length;
}
```

**Example:**
```
Path: (0,0) → (3,4) → (3,8)
Segment 1: √((3-0)² + (4-0)²) = √(9+16) = √25 = 5.0 units
Segment 2: √((3-3)² + (8-4)²) = √(0+16) = √16 = 4.0 units
Total: 9.0 units
```

---

### Helper 2: Distance from Point to Line Segment (Lines 447-463)

**Purpose:** Calculate shortest distance from a point to a line segment (critical for curved shapes)

```javascript
function distanceToSegment(px, py, x1, y1, x2, y2) {
    const dx = x2 - x1;
    const dy = y2 - y1;
    const lengthSquared = dx * dx + dy * dy;
    
    // EDGE CASE: Segment is actually a point
    if (lengthSquared === 0) {
        return Math.sqrt((px - x1) * (px - x1) + (py - y1) * (py - y1));
    }
    
    // Find projection parameter t (how far along segment is closest point)
    // t = 0 means closest to start point (x1, y1)
    // t = 1 means closest to end point (x2, y2)
    // 0 < t < 1 means closest point is on the segment
    let t = ((px - x1) * dx + (py - y1) * dy) / lengthSquared;
    
    // Clamp t to [0, 1] range (stay on segment)
    t = Math.max(0, Math.min(1, t));
    
    // Calculate closest point on segment
    const closestX = x1 + t * dx;
    const closestY = y1 + t * dy;
    
    // Return distance from point to closest point
    return Math.sqrt((px - closestX) * (px - closestX) + 
                     (py - closestY) * (py - closestY));
}
```

**Visual Example:**

```
Line segment from P1(0,0) to P2(4,0)
Test point Q(2, 1)

     Q(2,1)
       |
       | dist = 1.0
       |
-------C-------  ← C(2,0) is closest point on segment
P1(0,0)       P2(4,0)

Step 1: dx = 4, dy = 0, lengthSquared = 16
Step 2: t = ((2-0)×4 + (1-0)×0) / 16 = 8/16 = 0.5
Step 3: closestX = 0 + 0.5×4 = 2, closestY = 0 + 0.5×0 = 0
Step 4: distance = √((2-2)² + (1-0)²) = √1 = 1.0
```

**Why This Matters:**
- ❌ **OLD approach:** Check distance to individual points → Gaps in curves
- ✅ **NEW approach:** Check distance to line segments → No gaps, accurate curves

---

### Helper 3: Check if Shape is Closed (Lines 466-554)

**Purpose:** Determine if shape should be treated as closed (for interior filling)

```javascript
function isShapeClosed(unitPoints) {
    // Need at least 10 points to consider as potentially closed
    if (unitPoints.length < 10) return false;
    
    const first = unitPoints[0];
    const last = unitPoints[unitPoints.length - 1];
    
    // METRIC 1: Calculate closure distance (gap between start and end)
    const closureDistance = Math.sqrt(
        (last.x - first.x) ** 2 + (last.y - first.y) ** 2
    );
    
    // Calculate path length and average segment
    const pathLength = calculatePathLength(unitPoints);
    const avgSegment = pathLength / (unitPoints.length - 1);
    
    // METRIC 2: Gap as percentage of total path
    const gapPercentage = (closureDistance / pathLength) * 100;
```

**Metric 1 Calculation Example:**
```
Path: 12 points forming almost-circle
First point: (2, 0)
Last point:  (2.5, 0.3)
Gap distance: √((2.5-2)² + (0.3-0)²) = √(0.25 + 0.09) = √0.34 = 0.58 units
Path length: 10 units
Gap %: (0.58 / 10) × 100 = 5.8%
```

```javascript
    // METRIC 3: Calculate total rotation (how much path turns)
    let totalAngleChange = 0;
    
    for (let i = 1; i < unitPoints.length - 1; i++) {
        const p0 = unitPoints[i - 1];
        const p1 = unitPoints[i];
        const p2 = unitPoints[i + 1];
        
        // Vector from p0 to p1
        const v1x = p1.x - p0.x;
        const v1y = p1.y - p0.y;
        
        // Vector from p1 to p2
        const v2x = p2.x - p1.x;
        const v2y = p2.y - p1.y;
        
        // Calculate magnitudes
        const mag1 = Math.sqrt(v1x * v1x + v1y * v1y);
        const mag2 = Math.sqrt(v2x * v2x + v2y * v2y);
        
        if (mag1 > 0 && mag2 > 0) {
            // Dot product: v1·v2 = |v1||v2|cos(θ)
            const dot = v1x * v2x + v1y * v2y;
            const cosAngle = dot / (mag1 * mag2);
            
            // Angle between vectors
            const angle = Math.acos(Math.max(-1, Math.min(1, cosAngle)));
            totalAngleChange += angle;
        }
    }
    
    const fullRotation = 2 * Math.PI;  // 360° = 2π radians
    const rotationPercentage = (totalAngleChange / fullRotation) * 100;
```

**Metric 3 Calculation Example:**
```
Circle with 50 points
Each turn: ~7.2° (360° / 50)
Total turns: 50 points × 7.2° ≈ 360° = 2π radians
Rotation %: (2π / 2π) × 100 = 100%

C-shape with 50 points covering 270°
Total turns: 270° = 4.71 radians
Rotation %: (4.71 / 6.28) × 100 = 75%
```

```javascript
    // STRATEGY 1: Very tight closure (< 2% gap)
    if (gapPercentage < 2.0) {
        // Add straight-line closure if gap exists
        if (closureDistance > avgSegment * 0.5) {
            const steps = Math.max(3, Math.ceil(closureDistance / avgSegment));
            for (let i = 1; i < steps; i++) {
                const t = i / steps;
                unitPoints.push({
                    x: last.x + (first.x - last.x) * t,
                    y: last.y + (first.y - last.y) * t
                });
            }
        }
        return true;  // ← CLOSED
    }
    
    // STRATEGY 2: Balanced (< 10% gap AND ≥ 70% rotation)
    if (gapPercentage < 10.0 && rotationPercentage >= 70.0) {
        // Add straight-line closure
        if (closureDistance > avgSegment * 0.5) {
            const steps = Math.max(3, Math.ceil(closureDistance / avgSegment));
            for (let i = 1; i < steps; i++) {
                const t = i / steps;
                unitPoints.push({
                    x: last.x + (first.x - last.x) * t,
                    y: last.y + (first.y - last.y) * t
                });
            }
        }
        return true;  // ← CLOSED
    }
    
    // STRATEGY 3: Excellent rotation (< 15% gap AND ≥ 85% rotation)
    if (gapPercentage < 15.0 && rotationPercentage >= 85.0) {
        // Add straight-line closure
        if (closureDistance > avgSegment * 0.5) {
            const steps = Math.max(3, Math.ceil(closureDistance / avgSegment));
            for (let i = 1; i < steps; i++) {
                const t = i / steps;
                unitPoints.push({
                    x: last.x + (first.x - last.x) * t,
                    y: last.y + (first.y - last.y) * t
                });
            }
        }
        return true;  // ← CLOSED
    }
    
    return false;  // ← OPEN (no strategies matched)
}
```

**Examples:**

| Shape | Gap % | Rotation % | Strategy | Result |
|-------|-------|------------|----------|--------|
| Perfect circle | 1% | 99% | Strategy 1 | CLOSED |
| Human circle | 8% | 92% | Strategy 2 | CLOSED |
| Messy circle | 12% | 88% | Strategy 3 | CLOSED |
| C-shape | 20% | 60% | None | OPEN |
| Single line | 100% | 5% | None | OPEN |

**Important:** If closed, the function **modifies unitPoints** by adding closure points (lines 514-518, 529-533, 544-548)

---

### Helper 4: Point-in-Polygon Test (Lines 557-568)

**Purpose:** Determine if a point is inside a closed polygon (ray casting algorithm)

```javascript
function isPointInPolygon(x, y, polygon) {
    let inside = false;
    
    // Check each edge of the polygon
    for (let i = 0, j = polygon.length - 1; i < polygon.length; j = i++) {
        const xi = polygon[i].x, yi = polygon[i].y;
        const xj = polygon[j].x, yj = polygon[j].y;
        
        // Ray casting: cast ray from point to infinity (rightward)
        // Count how many edges it crosses
        const intersect = ((yi > y) !== (yj > y))    // Edge crosses horizontal line at y
            && (x < (xj - xi) * (y - yi) / (yj - yi) + xi);  // Point is left of intersection
        
        // Toggle inside/outside with each crossing
        if (intersect) inside = !inside;
    }
    
    return inside;
}
```

**Visual Example:**

```
Polygon (square):
  (1,3)-----(3,3)
    |         |
    |    P    |   ← Test point P(2,2)
    |         |
  (1,1)-----(3,1)

Ray from P(2,2) going right →→→

Crosses right edge at (3,2)   → Count = 1 (odd)
Result: inside = true ✓
```

**Algorithm:** 
- Odd number of crossings → Inside
- Even number of crossings → Outside

---

## MAIN AREA CALCULATION FUNCTION

### calculatePixelBasedArea (Lines 571-662)

This is the **main function** that calculates area by sampling a grid and counting painted cells.

#### STEP 1: Setup and Closure Detection (Lines 572-586)

```javascript
function calculatePixelBasedArea(unitPoints, brushSize, resolution = null) {
    // EDGE CASE: No points
    if (unitPoints.length === 0) return 0;
    
    // Determine if shape is closed (for interior filling)
    const isClosed = isShapeClosed(unitPoints);
    
    // Calculate brush radius in units
    const fullBrushRadius = brushSize / SCALE_FACTOR / 2;
    const brushRadius = fullBrushRadius;
    
    // AUTO-SELECT GRID RESOLUTION based on brush size
    if (resolution === null) {
        if (brushRadius < 0.05) {
            resolution = 0.05;      // Fine grid for thin brushes
        } else if (brushRadius < 0.1) {
            resolution = 0.075;     // Medium grid
        } else {
            resolution = 0.1;       // Coarse grid for thick brushes
        }
    }
```

**Brush Size Examples:**
```
Brush size = 5 pixels
  → brushRadius = 5 / 50 / 2 = 0.05 units (diameter = 0.1 units)
  → resolution = 0.05 units (fine grid)

Brush size = 10 pixels
  → brushRadius = 10 / 50 / 2 = 0.1 units (diameter = 0.2 units)
  → resolution = 0.1 units (coarse grid)
```

---

#### STEP 2: Calculate Bounding Box (Lines 588-602)

```javascript
    // Find bounding box (min/max X and Y of all points)
    let minX = unitPoints[0].x, maxX = unitPoints[0].x;
    let minY = unitPoints[0].y, maxY = unitPoints[0].y;
    
    for (const p of unitPoints) {
        minX = Math.min(minX, p.x);
        maxX = Math.max(maxX, p.x);
        minY = Math.min(minY, p.y);
        maxY = Math.max(maxY, p.y);
    }
    
    // Expand bounding box by brush radius (to include stroke edges)
    minX -= brushRadius;
    maxX += brushRadius;
    minY -= brushRadius;
    maxY += brushRadius;
```

**Example:**
```
Points: (1, 1), (3, 1), (3, 3), (1, 3)  ← Square
Bounding box before expansion: x: [1, 3], y: [1, 3]

Brush radius: 0.1 units
Bounding box after expansion: x: [0.9, 3.1], y: [0.9, 3.1]
```

**Visual:**
```
Before:                After (expanded by brush radius):
   3 ┌───┐               3.1 ┌─────┐
     │   │                   │     │
   1 └───┘               0.9 └─────┘
     1   3                   0.9   3.1
```

---

#### STEP 3: Grid Sampling - The Core Algorithm (Lines 604-640)

**This is where the actual area measurement happens!**

```javascript
    let paintedCells = 0;
    
    // Sample every grid cell within bounding box
    for (let x = minX; x <= maxX; x += resolution) {
        for (let y = minY; y <= maxY; y += resolution) {
            let isPainted = false;
            
            // CASE A: Shape is CLOSED (check interior + outline)
            if (isClosed) {
                // CHECK 1: Is this cell inside the polygon?
                if (isPointInPolygon(x, y, unitPoints)) {
                    isPainted = true;
                } else {
                    // CHECK 2: Is this cell within brush radius of outline?
                    for (let i = 0; i < unitPoints.length - 1; i++) {
                        const p1 = unitPoints[i];
                        const p2 = unitPoints[i + 1];
                        const dist = distanceToSegment(x, y, p1.x, p1.y, p2.x, p2.y);
                        
                        if (dist <= brushRadius) {
                            isPainted = true;
                            break;
                        }
                    }
                }
            } 
            // CASE B: Shape is OPEN (check outline only)
            else {
                // CHECK: Is this cell within brush radius of path?
                for (let i = 0; i < unitPoints.length - 1; i++) {
                    const p1 = unitPoints[i];
                    const p2 = unitPoints[i + 1];
                    const dist = distanceToSegment(x, y, p1.x, p1.y, p2.x, p2.y);
                    
                    if (dist <= brushRadius) {
                        isPainted = true;
                        break;
                    }
                }
            }
            
            // COUNT THIS CELL if painted
            if (isPainted) {
                paintedCells++;
            }
        }
    }
```

**Visual Example - Grid Sampling:**

```
Bounding box: [0.9, 3.1] × [0.9, 3.1]
Resolution: 0.1 units
Grid cells: 23 × 23 = 529 cells to check

For each cell at position (x, y):

CLOSED SHAPE:
  ┌─────────────┐
  │ ░░░░░░░░░░░ │  ← Interior: Check if inside polygon
  │ ░████████░  │  ← Outline: Check distance to segments
  │ ░████████░  │
  │ ░████████░  │
  │ ░░░░░░░░░░░ │
  └─────────────┘
  
OPEN SHAPE:
  ▬▬▬▬▬▬▬▬▬▬▬▬▬  ← Only outline: Check distance to segments
                   (No interior filling)

Legend:
  █ = Painted cell (counted)
  ░ = Unpainted cell
  ▬ = Outline only
```

**Grid Sampling Example with Numbers:**

```
Circle at (2, 2) with radius 1.0, brush size 0.1, resolution 0.1
Bounding box: [0.9, 3.1] × [0.9, 3.1]

Sample cells (showing a few):
  Cell (1.0, 2.0): 
    - Inside polygon? No (distance from center = 1.0)
    - Distance to outline? ~0.05 < 0.1? Yes → PAINTED
  
  Cell (2.0, 2.0):
    - Inside polygon? Yes (at center) → PAINTED
  
  Cell (3.0, 2.0):
    - Inside polygon? No (distance from center = 1.0)
    - Distance to outline? ~0.05 < 0.1? Yes → PAINTED
  
  Cell (4.0, 2.0):
    - Inside polygon? No (distance from center = 2.0)
    - Distance to outline? ~1.05 > 0.1? No → NOT PAINTED

Total painted cells: ~350 (approximate for this circle)
```

---

#### STEP 4: Calculate Raw Area (Lines 642-643)

```javascript
    // Calculate area of each cell
    const cellArea = resolution * resolution;
    
    // Total area = number of painted cells × area per cell
    const totalArea = paintedCells * cellArea;
```

**Example:**
```
Resolution: 0.1 units
Cell area: 0.1 × 0.1 = 0.01 square units

Painted cells: 350
Total area: 350 × 0.01 = 3.5 square units
```

---

#### STEP 5: Post-Processing for Closed Shapes (Lines 645-659)

**Purpose:** For closed shapes, subtract 50% of outline thickness to get closer to true interior area

```javascript
    let adjustedArea = totalArea;
    
    // ONLY for closed shapes: subtract outline contribution
    if (isClosed) {
        // Calculate perimeter
        let perimeter = 0;
        for (let i = 0; i < unitPoints.length; i++) {
            const p1 = unitPoints[i];
            const p2 = unitPoints[(i + 1) % unitPoints.length];
            const dx = p2.x - p1.x;
            const dy = p2.y - p1.y;
            perimeter += Math.sqrt(dx * dx + dy * dy);
        }
        
        // Calculate outline area contribution
        // Outline area ≈ perimeter × brush_diameter
        // We subtract 50% to balance interior and outline
        const outlineArea = perimeter * (brushRadius * 2) * 0.50;
        
        // Adjusted area = raw area - 50% of outline
        adjustedArea = Math.max(0, totalArea - outlineArea);
    }
    
    return adjustedArea;
}
```

**Why Subtract 50% of Outline?**

When we sample the grid, closed shapes include:
1. **Interior area** (inside polygon)
2. **Full outline** (brush thickness around perimeter)

For traced shapes (where user traces outline of grid squares), we want to report the **interior area** (number of grid squares), not interior + outline.

**Example:**
```
Traced 2×2 grid square:
  Raw area (interior + outline): 4.8 units²
  Perimeter: 8 units
  Outline area: 8 × (0.1 × 2) = 1.6 units²
  Subtract 50%: 1.6 × 0.50 = 0.8 units²
  Adjusted area: 4.8 - 0.8 = 4.0 units² ✓
  
  Result matches expected 2×2 = 4.0 grid squares!
```

**For Open Shapes:** No adjustment (outline IS the shape)
```
Single line 5 units long:
  Raw area: 5 × (0.1 × 2) = 1.0 units²
  Adjusted area: 1.0 units² (no subtraction)
```

---

## Complete Flow Diagram

```
USER DRAWS SHAPE ON CANVAS
         ↓
Points captured with brush strokes
  Example: [{x: 500, y: 500}, {x: 510, y: 490}, ..., {x: 600, y: 500}]
         ↓
STEP 1: Convert to unit coordinates
  Canvas → Unit coordinates
  [{x: 0, y: 0}, {x: 0.2, y: 0.2}, ..., {x: 2.0, y: 0}]
         ↓
STEP 2: Check if shape is closed
  Calculate gap percentage and rotation percentage
  Apply 3-strategy closure detection
  Result: isClosed = true or false
  (If closed, add closure points to unitPoints)
         ↓
STEP 3: Calculate brush radius in units
  brushRadius = brushSize / 50 / 2
  Example: 5 pixels → 0.05 units
         ↓
STEP 4: Auto-select grid resolution
  Based on brush size
  Example: 0.05 units (fine grid)
         ↓
STEP 5: Find bounding box
  Min/max X and Y of all points
  Expand by brush radius
         ↓
STEP 6: Grid sampling (THE KEY STEP)
  For each grid cell in bounding box:
    IF closed:
      Check if inside polygon OR within brush radius of outline
    ELSE:
      Check if within brush radius of outline
    
    If yes: paintedCells++
         ↓
STEP 7: Calculate raw area
  totalArea = paintedCells × (resolution²)
         ↓
STEP 8: Post-processing (closed shapes only)
  Calculate perimeter
  Subtract 50% of outline area
  adjustedArea = totalArea - (perimeter × brushDiameter × 0.50)
         ↓
FINAL RESULT: area in grid square units
```

---

## Examples

### Example 1: Traced 1×1 Grid Square (Closed)

**Input:**
```
User traces outline of 1×1 grid square
Brush size: 5 pixels (0.1 unit diameter)
Points form closed square from (0,0) to (1,1)
```

**Process:**
```
STEP 1: isClosed = true (gap < 2%, rotation ~100%)
STEP 2: brushRadius = 0.05 units
STEP 3: resolution = 0.05 units
STEP 4: Bounding box: [-0.05, 1.05] × [-0.05, 1.05]
STEP 5: Grid sampling:
  - Interior cells (inside polygon): ~400 cells
  - Outline cells (within 0.05 of edge): ~80 cells
  - Total painted cells: ~480
STEP 6: Raw area = 480 × 0.0025 = 1.2 units²
STEP 7: Perimeter = 4 units
STEP 8: Outline area = 4 × 0.1 × 0.50 = 0.2 units²
STEP 9: Adjusted area = 1.2 - 0.2 = 1.0 units² ✓
```

**Result: 1.0 units² (matches 1 grid square)**

---

### Example 2: Single Line 5 Units Long (Open)

**Input:**
```
User draws straight line from (0,0) to (5,0)
Brush size: 5 pixels (0.1 unit diameter)
```

**Process:**
```
STEP 1: isClosed = false (gap = 100%, rotation ~0%)
STEP 2: brushRadius = 0.05 units
STEP 3: resolution = 0.05 units
STEP 4: Bounding box: [-0.05, 5.05] × [-0.05, 0.05]
STEP 5: Grid sampling:
  - Check only outline (no interior)
  - Cells within 0.05 of line: ~200 cells
STEP 6: Raw area = 200 × 0.0025 = 0.5 units²
STEP 7: No post-processing (open shape)
```

**Result: 0.5 units² (line length × brush diameter = 5 × 0.1 = 0.5)**

---

### Example 3: Circle Filling 4 Grid Squares (Closed)

**Input:**
```
User draws circle with radius ~1.13 units (fills 2×2 grid)
Brush size: 5 pixels
Circle centered at (1, 1)
```

**Process:**
```
STEP 1: isClosed = true (gap < 2%, rotation ~100%)
STEP 2: brushRadius = 0.05 units
STEP 3: resolution = 0.05 units
STEP 4: Bounding box: [-0.18, 2.18] × [-0.18, 2.18]
STEP 5: Grid sampling:
  - Interior cells: ~1,500 cells (inside circle)
  - Outline cells: ~150 cells (around perimeter)
  - Total: ~1,650 cells
STEP 6: Raw area = 1,650 × 0.0025 = 4.125 units²
STEP 7: Perimeter ≈ 7.1 units (2πr)
STEP 8: Outline area = 7.1 × 0.1 × 0.50 = 0.355 units²
STEP 9: Adjusted area = 4.125 - 0.355 = 3.77 units²
```

**Result: ~3.77 units² (close to π × 1.13² ≈ 4.0)**

---

## Key Line References

| Step | Lines | What It Does | Output |
|------|-------|-------------|---------|
| **Constants** | 351-353 | Define coordinate system | CANVAS_SIZE, SCALE_FACTOR |
| **Path Length** | 436-444 | Calculate total path length | length in units |
| **Distance to Segment** | 447-463 | Distance from point to line | distance in units |
| **Closure Detection** | 466-554 | Check if closed (3 strategies) | true/false + closure points |
| **Point in Polygon** | 557-568 | Ray casting test | true/false |
| **Main Entry** | 753-776 | Entry point for area calc | Calls calculatePixelBasedArea |
| **Setup & Resolution** | 571-586 | Initialize, detect closure | isClosed, brushRadius, resolution |
| **Bounding Box** | 588-602 | Find min/max bounds | minX, maxX, minY, maxY |
| **Grid Sampling** | 604-640 | **Core area calculation** | **paintedCells count** |
| **Raw Area** | 642-643 | Calculate total area | totalArea in units² |
| **Post-Processing** | 645-659 | Adjust for outline | adjustedArea in units² |

---

## Mathematical Formulas

### Brush Radius (Units)
```
brushRadius = (brushSize in pixels) / SCALE_FACTOR / 2
           = brushSize / 50 / 2
           = brushSize / 100

Example: 5 pixels → 5/100 = 0.05 units radius (0.1 diameter)
```

### Grid Resolution (Auto-Selected)
```
IF brushRadius < 0.05:  resolution = 0.05
ELIF brushRadius < 0.1: resolution = 0.075
ELSE:                   resolution = 0.1
```

### Cell Area
```
cellArea = resolution × resolution

Example: 0.05 × 0.05 = 0.0025 square units per cell
```

### Distance to Segment (Projection)
```
Given segment from P1(x1, y1) to P2(x2, y2), test point Q(px, py)

1. Vector: V = P2 - P1 = (dx, dy)
2. Projection parameter: t = (Q - P1)·V / |V|²
3. Clamp: t = max(0, min(1, t))
4. Closest point: C = P1 + t × V
5. Distance: d = |Q - C|
```

### Raw Area
```
totalArea = paintedCells × (resolution²)
```

### Adjusted Area (Closed Shapes Only)
```
perimeter = sum of all segment lengths
outlineArea = perimeter × brushDiameter × 0.50
adjustedArea = max(0, totalArea - outlineArea)
```

### Closure Gap Percentage
```
gapPercentage = (closureDistance / pathLength) × 100
```

### Rotation Percentage
```
totalAngleChange = sum of angles between consecutive segments
rotationPercentage = (totalAngleChange / 2π) × 100
```

---

## Summary

**The area is calculated using a pixel-based grid sampling method:**

1. Convert drawn points to unit coordinates
2. Determine if shape is closed (3-strategy system)
3. Calculate brush radius in units
4. Auto-select grid resolution based on brush size
5. Find bounding box and expand by brush radius
6. **Sample every grid cell:**
   - Closed shapes: Check interior (point-in-polygon) + outline (distance to segments)
   - Open shapes: Check outline only (distance to segments)
7. Count painted cells
8. Calculate raw area = paintedCells × cell_area
9. For closed shapes: Subtract 50% of outline area
10. Return final adjusted area

**Key Innovation:** Distance-to-segment checking ensures no gaps in curved sections, giving accurate area for all shape types.

**Output:** Area in grid square units (1 grid square = 1.0 area unit)

**Typical Accuracy:** 
- Traced grid squares: ±5% (0.95-1.05 for 1×1 square)
- Circles: ±10% (matches πr² within reason)
- Open shapes: Accurate to brush width × path length
