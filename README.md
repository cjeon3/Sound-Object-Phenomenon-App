# Sound Object Visualization Research Tool

## Technical Documentation

**Version:** 2.2  
**Institution:** UCI Hearing & Speech Lab  
**Application:** Acoustic Perception Research (Sound Object Phenomenon Study)  
**Last Revised:** November 2025

---

## 1. Introduction

This document describes the computational methods implemented in the Sound Object Visualization Research Tool, a web-based application designed to capture and analyze graphical representations of auditory spatial perception. The tool enables participants to draw shapes corresponding to their perception of sound objects at various frequencies and intensities, with subsequent geometric analysis of the resulting forms.

The primary outputs are area measurements and centroid coordinates for each drawn shape, computed using algorithms selected to address the particular challenges of hand-drawn input data.

---

## 2. System Specifications

### 2.1 Canvas and Coordinate System

The drawing canvas employs a Cartesian coordinate system with the following parameters:

| Parameter | Value | Rationale |
|-----------|-------|-----------|
| Canvas dimensions | 1000 × 1000 pixels | Provides sufficient resolution for detailed drawings while maintaining computational efficiency |
| Coordinate range | −10 to +10 units (both axes) | Symmetric range centered on origin facilitates interpretation of lateralization data |
| Scale factor | 50 pixels per unit | Derived from CANVAS_SIZE / (UNIT_RANGE × 2) = 1000 / 20 |
| Grid resolution | 1.0 units per grid line | Grid lines at every 2 units with labels; one grid square equals 1.0 square unit |
| Reference circle | 3-unit radius | Provides consistent spatial reference across trials |

### 2.2 Frequency and Intensity Conditions

The tool supports 11 frequency-intensity combinations:

| Frequency (Hz) | Intensity (dB SPL) |
|----------------|-------------------|
| 31 | 100 |
| 62.5 | 100 |
| 125 | 90 |
| 250 | 85 |
| 500 | 80 |
| 1000 | 80 |
| 2000 | 80 |
| 4000 | 80 |
| 8000 | 90 |
| 12000 | 85 |
| 16000 | 85 |

Each frequency maintains independent drawing data, undo/redo stacks, and analysis results.

### 2.3 Brush System

Drawing implements a circular brush with configurable diameter:

| Brush size (pixels) | Radius (units) | Calculation |
|---------------------|----------------|-------------|
| 1 | 0.010 | (1 / 50) / 2 |
| 5 | 0.050 | (5 / 50) / 2 |
| 10 | 0.100 | (10 / 50) / 2 |

The brush size range is 1–10 pixels. Brush thickness is incorporated into all area calculations to ensure that measured area reflects the visual footprint of the drawn shape.

---

## 3. Area Calculation Methodology

### 3.1 Rationale for Pixel-Based Approach

Traditional polygon area formulas (e.g., the shoelace formula) compute the area enclosed by a path's centerline, ignoring stroke width. For hand-drawn shapes where the visual appearance includes substantial brush thickness, this approach underestimates the perceived area. Additionally, shapes drawn as outlines (traced boundaries rather than filled regions) would yield minimal area despite enclosing substantial visual space.

The implemented method addresses these limitations through pixel-based sampling that measures the actual visual footprint, including brush thickness and interior regions of closed shapes.

### 3.2 Curve-Aware Distance Calculation

The implementation computes distance to line segments connecting consecutive path points rather than to discrete vertices. This formulation ensures that curved regions are properly captured.

The distance from point P to segment AB is computed as:

```
function distanceToSegment(px, py, x1, y1, x2, y2):
    dx = x2 - x1
    dy = y2 - y1
    lengthSquared = dx² + dy²
    
    if lengthSquared = 0:
        return √((px - x1)² + (py - y1)²)
    
    t = ((px - x1) × dx + (py - y1) × dy) / lengthSquared
    t = clamp(t, 0, 1)
    
    closestX = x1 + t × dx
    closestY = y1 + t × dy
    
    return √((px - closestX)² + (py - closestY)²)
```

### 3.3 Algorithm Description

The area calculation proceeds as follows:

1. **Closure detection**: Determine whether the shape should be treated as closed (see Section 4). If classified as closed, interpolation points may be added to close the gap between endpoints.

2. **Brush radius computation**: Calculate the effective brush radius in unit coordinates:
   ```
   brushRadius = brushSize / SCALE_FACTOR / 2
   ```

3. **Adaptive resolution selection**: Grid resolution is selected based on brush size:
   - Brush radius < 0.05 units: resolution = 0.05 units
   - Brush radius < 0.1 units: resolution = 0.075 units
   - Otherwise: resolution = 0.1 units

4. **Bounding box construction**: Determine the axis-aligned bounding box of all path points, expanded by the brush radius in all directions.

5. **Pixel classification**: For each grid cell within the bounding box, determine membership by two criteria:
   - **Interior containment** (closed shapes only): The cell center lies within the polygon boundary, determined by ray casting.
   - **Stroke proximity**: The cell center lies within brush radius of any line segment in the path.

6. **Area computation**: Sum the areas of all classified cells:
   ```
   Area = (number of painted cells) × (resolution²)
   ```

7. **Outline correction** (closed shapes only): Subtract 50% of the estimated outline area to reduce double-counting where the stroke overlaps the filled interior:
   ```
   outlineArea = perimeter × (brushRadius × 2) × 0.50
   adjustedArea = max(0, totalArea - outlineArea)
   ```

### 3.4 Interior Detection via Ray Casting

Point-in-polygon testing employs the standard ray casting algorithm:

```
function isPointInPolygon(x, y, polygon):
    inside = false
    for each edge (i, j) in polygon:
        xi, yi = polygon[i]
        xj, yj = polygon[j]
        
        if ((yi > y) ≠ (yj > y)) and 
           (x < (xj - xi) × (y - yi) / (yj - yi) + xi):
            inside = not inside
    
    return inside
```

This method handles self-intersecting paths and concave shapes without special cases.

### 3.5 Treatment of Open Shapes

Unlike some implementations that return area = 0 for open shapes, this tool calculates the stroke area (ribbon area) for all shapes regardless of closure status. For open shapes, only the stroke proximity criterion applies; interior filling is not performed. This design decision ensures that all drawn content contributes to the measured area, reflecting the visual footprint of the participant's drawing.

### 3.6 Computational Complexity

The algorithm exhibits O(n × m) complexity where n is the number of path segments and m is the number of grid cells. For typical shapes, execution is negligible relative to user interaction time.

---

## 4. Closure Detection Methodology

### 4.1 Problem Statement

Determining whether a hand-drawn shape represents a closed region presents a classification challenge. Strict geometric criteria (requiring exact endpoint coincidence) reject nearly all human-drawn shapes due to motor variability. Conversely, overly permissive criteria incorrectly classify intentionally open shapes (C-shapes, arcs, single strokes) as closed.

The implemented solution employs a three-strategy system that balances sensitivity to intentional closure against tolerance for natural drawing imprecision.

### 4.2 Metrics

Two metrics characterize potential closure:

**Gap percentage**: The Euclidean distance between path endpoints, expressed as a percentage of total path length.
```
closureDistance = √((last.x - first.x)² + (last.y - first.y)²)
gapPercentage = (closureDistance / pathLength) × 100
```

**Rotation percentage**: The cumulative angular change along the path, expressed as a percentage of a full rotation (2π radians).

```
For each triple of consecutive points (p0, p1, p2):
    v1 = (p1.x - p0.x, p1.y - p0.y)
    v2 = (p2.x - p1.x, p2.y - p1.y)
    
    cosAngle = (v1 · v2) / (|v1| × |v2|)
    angle = arccos(clamp(cosAngle, -1, 1))
    totalAngleChange += angle

rotationPercentage = (totalAngleChange / 2π) × 100
```

### 4.3 Classification Strategies

A shape is classified as closed if it satisfies any of the following criteria:

**Strategy 1 (Tight closure)**:
```
gapPercentage < 2.0%
```
Rationale: Very small gaps relative to path length indicate intentional closure regardless of rotation. This captures nearly-perfect circles and loops.

**Strategy 2 (Balanced)**:
```
gapPercentage < 10.0% AND rotationPercentage ≥ 70.0%
```
Rationale: Moderate gaps are acceptable when accompanied by substantial rotation (at least 252° of the full circle). This represents the most common case for human-drawn closed shapes with imperfect endpoints.

**Strategy 3 (High rotation compensation)**:
```
gapPercentage < 15.0% AND rotationPercentage ≥ 85.0%
```
Rationale: Larger gaps may result from motor noise at the end of an otherwise complete rotation. Near-complete rotation (306°+) compensates for larger endpoint discrepancy.

### 4.4 Gap Interpolation

When a shape is classified as closed but the endpoint gap exceeds half the average segment length, interpolation points are added to close the path:

```
if closureDistance > avgSegment × 0.5:
    steps = max(3, ceil(closureDistance / avgSegment))
    for i = 1 to steps - 1:
        t = i / steps
        interpolatedPoint = last + t × (first - last)
        append interpolatedPoint to path
```

This ensures smooth closure for area calculation and interior detection.

### 4.5 Minimum Point Threshold

Shapes with fewer than 10 recorded points are automatically classified as open. This threshold prevents spurious closure detection from very brief strokes where gap and rotation metrics are unreliable.

### 4.6 Classification Outcomes by Shape Type

| Shape | Typical Gap | Typical Rotation | Classification |
|-------|-------------|------------------|----------------|
| Precise circle | < 2% | 95–100% | Closed (Strategy 1) |
| Hand-drawn circle | 3–8% | 75–95% | Closed (Strategy 2) |
| Imprecise loop | 8–12% | 85–95% | Closed (Strategy 2 or 3) |
| Closed horseshoe | 5–10% | 72–80% | Closed (Strategy 2) |
| C-shape | 15–25% | 50–65% | Open |
| Half circle | 30–50% | 40–50% | Open |
| Single line | 40–100% | 0–30% | Open |

---

## 5. Centroid Calculation Methodology

### 5.1 Approach

The centroid is computed as the geometric center of uniformly resampled points along the path. This approach addresses the bias introduced by variable drawing speed: regions drawn slowly accumulate more recorded points than regions drawn quickly, which would skew a simple arithmetic mean toward slowly-drawn segments.

### 5.2 Uniform Resampling

To eliminate drawing-speed bias, points are resampled at uniform intervals along the path length:

```
1. Calculate total path length by summing segment lengths
2. Determine number of samples: max(100, floor(totalLength / 0.1))
3. Calculate sample interval: totalLength / numSamples
4. For each sample point:
   - Find the segment containing the target distance
   - Interpolate position within that segment
   - Add to resampled point set
```

### 5.3 Centroid Computation

The centroid is the arithmetic mean of all resampled points:

```
sumX = Σ(resampledPoints[i].x)
sumY = Σ(resampledPoints[i].y)

centroidX = sumX / numResampledPoints
centroidY = sumY / numResampledPoints
```

### 5.4 Coordinate Transformation

Path points are recorded in canvas coordinates (pixels) and converted to unit coordinates for centroid calculation:

```
function canvasToUnit(x, y):
    centerX = CANVAS_SIZE / 2
    centerY = CANVAS_SIZE / 2
    return {
        x: (x - centerX) / SCALE_FACTOR,
        y: (centerY - y) / SCALE_FACTOR
    }
```

Note that the y-axis is inverted (canvas y increases downward; unit y increases upward).

---

## 6. Data Export Specifications

### 6.1 CSV Format

Exported analysis data follows this structure:

```
Participant,Frequency_Hz,Frequency_dB,Shape_ID,Color_Hex,Area_Grid_Squares,Centroid_X,Centroid_Y,Total_Points,Raw_Path_Coordinates
P-001,125,90,1,#ef4444,12.4567,1.2345,-0.5678,156,"0.123,0.456|0.234,0.567|..."
```

Fields:
- **Participant**: Participant identifier
- **Frequency_Hz**: Stimulus frequency in Hertz
- **Frequency_dB**: Stimulus intensity in dB SPL
- **Shape_ID**: Sequential shape number within this frequency
- **Color_Hex**: Drawing color as hexadecimal code
- **Area_Grid_Squares**: Computed area in square units
- **Centroid_X, Centroid_Y**: Centroid coordinates in unit space
- **Total_Points**: Number of recorded path points
- **Raw_Path_Coordinates**: Pipe-separated list of (x,y) coordinate pairs in unit space

### 6.2 Image Export

PNG images are exported at native canvas resolution (1000 × 1000 pixels) with the following elements rendered:
- Background image (if uploaded)
- Reference circle (3-unit radius)
- Coordinate grid with axis labels
- All drawn shapes with original colors and brush sizes
- Centroid markers (cross within circle) for each shape

Filename convention: `ParticipantID_FrequencyHz_IntensitydB.png`

### 6.3 Archive Format

Complete exports are packaged as ZIP archives containing one PNG per frequency that has drawings. Archive naming follows: `ParticipantID_##_Drawings.zip` where ## is a chronological counter (persisted in localStorage) ensuring unique filenames across export sessions.

### 6.4 Google Integration

The tool supports direct export to Google Drive (ZIP upload) and Google Sheets (data rows) via Google Apps Script Web App endpoints. URLs are persisted in localStorage for convenience.

---

## 7. Algorithm Parameters

The following parameters are defined in the implementation:

### 7.1 Canvas Configuration

| Parameter | Value | Constant Name |
|-----------|-------|---------------|
| Canvas size | 1000 pixels | CANVAS_SIZE |
| Unit range | 10 units (−10 to +10) | UNIT_RANGE |
| Scale factor | 50 pixels/unit | SCALE_FACTOR |
| Reference circle radius | 3 units | BACKGROUND_CIRCLE_RADIUS_UNITS |

### 7.2 Area Calculation

| Parameter | Value | Description |
|-----------|-------|-------------|
| Base resolution | 0.1 units | Default sampling grid cell size |
| Fine resolution | 0.05 units | Used when brush radius < 0.05 |
| Medium resolution | 0.075 units | Used when brush radius < 0.1 |
| Outline subtraction | 50% | Proportion of outline area subtracted for closed shapes |

### 7.3 Closure Detection

| Parameter | Value | Description |
|-----------|-------|-------------|
| Strategy 1 gap threshold | 2.0% | Maximum gap for unconditional closure |
| Strategy 2 gap threshold | 10.0% | Maximum gap with rotation requirement |
| Strategy 2 rotation threshold | 70.0% | Minimum rotation for Strategy 2 |
| Strategy 3 gap threshold | 15.0% | Maximum gap with high rotation |
| Strategy 3 rotation threshold | 85.0% | Minimum rotation for Strategy 3 |
| Minimum points | 10 | Below this, shapes are always classified as open |

### 7.4 Centroid Calculation

| Parameter | Value | Description |
|-----------|-------|-------------|
| Minimum samples | 100 | Minimum resampled points for centroid |
| Sample density | 0.1 units | Maximum distance between samples |

---

## 8. User Interface Elements

### 8.1 Drawing Controls

- **Color palette**: 7 colors (Red #ef4444, Orange #f97316, Yellow #facc15, Green #10b981, Blue #3b82f6, Purple #8b5cf6, Black #000000)
- **Brush size slider**: Range 1–10 pixels
- **Toggle drawing**: Enable/disable drawing mode
- **Clear current**: Clear shapes for current frequency only
- **Undo/Redo**: Per-frequency undo/redo stacks
- **Reset all**: Clear all frequencies and reset settings (preserves background image)

### 8.2 Background Image

An optional background image can be uploaded and displayed behind the coordinate grid. The image is scaled to fill the canvas and persists across frequency changes. Reset All preserves the background image.

---

## 9. Methodological Considerations for Research Use

### 9.1 Reproducibility

All algorithms are deterministic. Given identical input (recorded path coordinates, brush size, and parameter settings), outputs are exactly reproducible. No random number generation is employed in any calculation.

### 9.2 Limitations

1. Brush size is constant within each stroke. Variable-pressure input is not supported.
2. Self-intersecting paths are handled correctly for area calculation but may produce interior regions that the ray casting algorithm classifies inconsistently depending on intersection topology.
3. Very sparse shapes (< 10 points) are excluded from closure analysis due to unreliable metrics.
4. The 50% outline subtraction factor is empirically determined and may require adjustment for specific brush sizes or drawing styles.
5. The centroid may fall outside concave shapes (e.g., C-shapes) since it is computed as an arithmetic mean rather than a geometric median.

### 9.3 Suggested Methodology Description for Publications

> Participants drew shapes on a 20 × 20 unit Cartesian grid (1000 × 1000 pixels; 50 pixels per unit) using circular brushes with configurable diameter (1–10 pixels, corresponding to 0.02–0.20 unit diameters). Shape areas were computed using a curve-aware pixel-based method with adaptive grid resolution (0.05–0.1 units) that measured the visual footprint of strokes by computing minimum distance to line segments between consecutive path points. For closed shapes, interior areas were determined via ray casting point-in-polygon testing, with 50% outline subtraction applied to reduce double-counting of stroke area within the filled region.
>
> Shape closure was determined by a three-strategy classification system based on endpoint gap (as percentage of path length) and cumulative rotation (as percentage of 360°). Shapes meeting any of the following criteria were classified as closed: (1) gap < 2%; (2) gap < 10% and rotation ≥ 70%; or (3) gap < 15% and rotation ≥ 85%. 
>
> Centroid coordinates were computed as the arithmetic mean of points uniformly resampled along the path length, eliminating bias from variable drawing speed. A minimum of 100 samples or one sample per 0.1 units of path length (whichever was greater) ensured consistent centroid estimation across shapes of varying complexity.

---

## 10. References

Preparata, F. P., & Shamos, M. I. (1985). *Computational Geometry: An Introduction*. Springer-Verlag. [Point-in-polygon algorithms, distance to line segment computation]

---

## 11. Version History

| Version | Date | Changes |
|---------|------|---------|
| 2.2 | November 2025 | Curve-aware distance calculation; three-strategy closure detection; uniform path resampling for centroid |
| 2.1 | November 2025 | Pixel-based area with interior filling; multi-criteria closure detection |
| 2.0 | November 2025 | Adaptive area calculation methods |
| 1.5 | November 2025 | Google Drive and Sheets integration |
| 1.0 | 2024 | Initial release |

---

**UCI Hearing & Speech Lab**  
**Sound Object Visualization Research Tool v2.2**
