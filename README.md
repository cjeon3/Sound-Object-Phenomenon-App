# Sound Object Visualization Research Tool

## Technical Documentation

**Version:** 2.2  
**Institution:** UCI Hearing & Speech Lab  
**Application:** Acoustic Perception Research  
**Last Revised:** November 2025

---

## 1. Introduction

This document describes the computational methods implemented in the Sound Object Visualization Research Tool, a web-based application designed to capture and analyze graphical representations of auditory spatial perception. The tool enables participants to draw shapes corresponding to their perception of sound objects at various frequencies, with subsequent geometric analysis of the resulting forms.

The primary outputs are area measurements and centroid coordinates for each drawn shape, computed using algorithms selected to address the particular challenges of hand-drawn input data.

---

## 2. System Specifications

### 2.1 Canvas and Coordinate System

The drawing canvas employs a Cartesian coordinate system with the following parameters:

| Parameter | Value | Rationale |
|-----------|-------|-----------|
| Canvas dimensions | 600 × 600 pixels | Provides sufficient resolution for detailed drawings while maintaining computational efficiency |
| Coordinate range | −5 to +5 units (both axes) | Symmetric range centered on origin facilitates interpretation of lateralization data |
| Scale factor | 60 pixels per unit | Integer conversion simplifies coordinate transformations |
| Grid resolution | 0.1 units per grid line | One grid square equals 1.0 square units, enabling visual estimation of area |
| Reference circle | 3-unit radius | Provides consistent spatial reference across trials |

### 2.2 Brush System

Drawing implements a circular brush with configurable diameter:

| Brush size (pixels) | Radius (units) | Calculation |
|---------------------|----------------|-------------|
| 1 | 0.008 | (1 / 60) / 2 |
| 5 | 0.042 | (5 / 60) / 2 |
| 10 | 0.083 | (10 / 60) / 2 |
| 20 | 0.167 | (20 / 60) / 2 |

Brush thickness is incorporated into all area calculations to ensure that measured area reflects the visual footprint of the drawn shape.

---

## 3. Area Calculation Methodology

### 3.1 Rationale for Pixel-Based Approach

Traditional polygon area formulas (e.g., the shoelace formula) compute the area enclosed by a path's centerline, ignoring stroke width. For hand-drawn shapes where the visual appearance includes substantial brush thickness, this approach underestimates the perceived area. Additionally, shapes drawn as outlines (traced boundaries rather than filled regions) would yield minimal area despite enclosing substantial visual space.

The implemented method addresses these limitations through pixel-based sampling that measures the actual visual footprint, including brush thickness and interior regions of closed shapes.

### 3.2 Curve-Aware Distance Calculation

A critical refinement in version 2.2 addresses the handling of curved path segments. Earlier implementations computed the distance from each sampling point to discrete path vertices, which produced systematic errors in curved regions where the spacing between vertices exceeded the sampling resolution.

The current implementation computes distance to line segments connecting consecutive path points:

```
For point P and segment defined by endpoints A and B:
    
    Let v = B - A (segment vector)
    Let w = P - A (vector from A to query point)
    
    Compute projection parameter:
        t = (w · v) / (v · v)
        t = clamp(t, 0, 1)
    
    Closest point on segment:
        C = A + t × v
    
    Distance = ||P - C||
```

This formulation ensures that curved regions are properly captured, eliminating gaps that previously occurred in crescent, horseshoe, and other concave shapes.

### 3.3 Algorithm Description

The area calculation proceeds as follows:

1. **Closure detection**: Determine whether the shape should be treated as closed (see Section 4).

2. **Open shape handling**: If the shape is classified as open, return area = 0. This prevents the computation of "ribbon area" (the stroke itself) for shapes that do not enclose a region, such as C-shapes or single lines.

3. **Grid generation**: Construct a sampling grid over the shape's bounding box. Grid resolution is adaptive based on brush size:
   - Brush radius < 0.05 units: resolution = 0.05 units
   - Brush radius < 0.1 units: resolution = 0.075 units
   - Otherwise: resolution = 0.1 units

4. **Pixel classification**: For each grid cell, determine membership by two criteria:
   - **Stroke proximity**: The cell center lies within brush radius of any line segment in the path (using the curve-aware distance calculation).
   - **Interior containment**: For closed shapes, the cell center lies within the polygon boundary (determined by ray casting).

5. **Area computation**: Sum the areas of all classified cells:
   ```
   Area = (number of marked cells) × (resolution²)
   ```

6. **Outline correction**: For closed shapes, subtract 80% of the estimated outline area to avoid double-counting the stroke in interior calculations:
   ```
   Corrected Area = Raw Area − (0.80 × Outline Area)
   ```

### 3.4 Interior Detection via Ray Casting

Point-in-polygon testing employs the standard ray casting algorithm:

```
Count intersections between a horizontal ray from point P 
and all edges of the polygon.

If intersection count is odd: P is inside.
If intersection count is even: P is outside.
```

This method handles self-intersecting paths and concave shapes without special cases.

### 3.5 Computational Complexity

The algorithm exhibits O(n × m) complexity where n is the number of path segments and m is the number of grid cells. For typical shapes, execution time ranges from 10–30 ms per shape, which is negligible relative to user interaction time.

---

## 4. Closure Detection Methodology

### 4.1 Problem Statement

Determining whether a hand-drawn shape represents a closed region presents a classification challenge. Strict geometric criteria (requiring exact endpoint coincidence) reject nearly all human-drawn shapes due to motor variability. Conversely, overly permissive criteria incorrectly classify intentionally open shapes (C-shapes, arcs, single strokes) as closed.

The implemented solution employs a three-strategy system that balances sensitivity to intentional closure against tolerance for natural drawing imprecision.

### 4.2 Metrics

Two metrics characterize potential closure:

**Gap percentage**: The Euclidean distance between path endpoints, expressed as a percentage of total path length.
```
Gap % = (||endpoint_first − endpoint_last|| / total_path_length) × 100
```

**Rotation percentage**: The cumulative angular displacement around the path, expressed as a percentage of 360°.
```
Rotation % = (Σ|Δθᵢ| / 360°) × 100
```

where Δθᵢ is the angular change between consecutive segments.

### 4.3 Classification Strategies

A shape is classified as closed if it satisfies any of the following criteria:

**Strategy 1 (Tight closure)**:
```
Gap percentage < 2.0%
```
Rationale: Very small gaps relative to path length indicate intentional closure regardless of rotation. This captures nearly-perfect circles and loops.

**Strategy 2 (Balanced)**:
```
Gap percentage < 10.0% AND Rotation percentage ≥ 70.0%
```
Rationale: Moderate gaps are acceptable when accompanied by substantial rotation (at least 252° of the full circle). This represents the most common case for human-drawn closed shapes with imperfect endpoints.

**Strategy 3 (High rotation compensation)**:
```
Gap percentage < 15.0% AND Rotation percentage ≥ 85.0%
```
Rationale: Larger gaps may result from motor noise at the end of an otherwise complete rotation. Near-complete rotation (306°+) compensates for larger endpoint discrepancy.

### 4.4 Classification Outcomes by Shape Type

| Shape | Typical Gap | Typical Rotation | Classification |
|-------|-------------|------------------|----------------|
| Precise circle | < 2% | 95–100% | Closed (Strategy 1) |
| Hand-drawn circle | 3–8% | 75–95% | Closed (Strategy 2) |
| Imprecise loop | 8–12% | 85–95% | Closed (Strategy 2 or 3) |
| Closed horseshoe | 5–10% | 72–80% | Closed (Strategy 2) |
| C-shape | 15–25% | 50–65% | Open |
| Half circle | 30–50% | 40–50% | Open |
| Single line | 40–100% | 0–30% | Open |

### 4.5 Minimum Point Threshold

Shapes with fewer than 10 recorded points are automatically classified as open. This threshold prevents spurious closure detection from very brief strokes where gap and rotation metrics are unreliable.

---

## 5. Centroid Calculation Methodology

### 5.1 Rationale for Geometric Median Approach

The conventional geometric centroid (arithmetic mean of coordinates) can fall outside the boundary of concave shapes. For a C-shape or horseshoe, the centroid often lies in empty space, rendering it meaningless as a representative location.

The geometric median (also termed the medoid or Fermat point) minimizes the sum of distances to all points in the shape and is guaranteed to lie within the convex hull of those points. For non-convex shapes, this property ensures the representative point falls on or within the drawn region.

### 5.2 Weiszfeld's Algorithm

The geometric median is computed iteratively using Weiszfeld's algorithm:

```
Initialize: x₀, y₀ = arithmetic mean of path points

Iterate until convergence (tolerance = 0.001 units):
    
    numerator_x = Σ(xᵢ / dᵢ)
    numerator_y = Σ(yᵢ / dᵢ)
    denominator = Σ(1 / dᵢ)
    
    where dᵢ = distance from current estimate to point i
    
    x_new = numerator_x / denominator
    y_new = numerator_y / denominator

Output: (x_final, y_final)
```

Convergence typically occurs within 5–15 iterations, requiring less than 5 ms per shape.

### 5.3 Adaptive Method Selection

The system selects among four centroid variants based on shape characteristics:

**Basic medoid**: Default method using Weiszfeld's algorithm on all path points.

**Skeleton-constrained medoid**: Applied when aspect ratio exceeds 3.0 (highly elongated shapes). The medoid is computed from uniformly sampled points along the path rather than all recorded points, preventing clustering artifacts from variable drawing speed.

**Density-weighted medoid**: Applied when point density variance exceeds 0.5 (indicating regions of varied stroke concentration). Points are weighted by inverse distance to path, emphasizing regions of greater visual mass.

**Brush-aware medoid**: Applied when brush size exceeds 15 pixels. Points are weighted by local neighbor density within the brush radius, accounting for the visual mass distribution of thick strokes.

### 5.4 Selection Criteria

```
If aspect_ratio > 3.0:
    Use skeleton-constrained medoid
Else if density_variance > 0.5:
    Use density-weighted medoid  
Else if brush_size > 15:
    Use brush-aware medoid
Else:
    Use basic medoid
```

---

## 6. Data Export Specifications

### 6.1 CSV Format

Exported analysis data follows this structure:

```
Participant,Frequency (Hz),Shape Number,Area (sq units),Centroid X,Centroid Y
P-001,125,1,12.4567,1.2345,-0.5678
P-001,125,2,8.9012,0.5432,2.1098
```

Area is expressed in square coordinate units (where one grid square = 1.0 square units). Centroid coordinates are in the same unit system as the drawing canvas (range −5 to +5).

### 6.2 Image Export

PNG images are exported at native canvas resolution (600 × 600 pixels) with the following elements rendered: coordinate grid, axis labels, all drawn shapes with original colors, and centroid markers.

Filename convention: `ParticipantID_FrequencyHz.png`

### 6.3 Archive Format

Complete exports are packaged as ZIP archives containing one PNG per frequency. Archive naming follows: `ParticipantID_##.zip` where ## is a chronological counter ensuring unique filenames.

---

## 7. Algorithm Parameters

The following parameters may be adjusted for specific research requirements:

### 7.1 Area Calculation

| Parameter | Default | Description |
|-----------|---------|-------------|
| Base resolution | 0.1 units | Sampling grid cell size |
| Fine resolution | 0.05 units | Used for small brush sizes |
| Outline subtraction | 80% | Proportion of outline area subtracted from closed shapes |

### 7.2 Closure Detection

| Parameter | Default | Description |
|-----------|---------|-------------|
| Strategy 1 gap threshold | 2.0% | Maximum gap for unconditional closure |
| Strategy 2 gap threshold | 10.0% | Maximum gap with rotation requirement |
| Strategy 2 rotation threshold | 70.0% | Minimum rotation for Strategy 2 |
| Strategy 3 gap threshold | 15.0% | Maximum gap with high rotation |
| Strategy 3 rotation threshold | 85.0% | Minimum rotation for Strategy 3 |
| Minimum points | 10 | Below this, shapes are always classified as open |

### 7.3 Centroid Calculation

| Parameter | Default | Description |
|-----------|---------|-------------|
| Convergence tolerance | 0.001 units | Stopping criterion for Weiszfeld iteration |
| Maximum iterations | 50 | Safety limit to prevent non-convergence |
| Elongation threshold | 3.0 | Aspect ratio triggering skeleton-constrained method |
| Density variance threshold | 0.5 | Variance triggering density-weighted method |
| Large brush threshold | 15 pixels | Brush size triggering brush-aware method |

---

## 8. Methodological Considerations for Research Use

### 8.1 Reproducibility

All algorithms are deterministic. Given identical input (recorded path coordinates, brush size, and parameter settings), outputs are exactly reproducible. No random number generation is employed in any calculation.

### 8.2 Validation

The curve-aware distance calculation eliminates systematic underestimation of area in curved regions that affected earlier versions. The three-strategy closure detection correctly classifies intentionally open shapes (C-shapes, lines, arcs) as open while accommodating the endpoint imprecision characteristic of hand-drawn closed shapes.

### 8.3 Limitations

1. Brush size is constant within each stroke. Variable-pressure input is not supported.
2. Self-intersecting paths are handled correctly for area calculation but may produce interior regions that the ray casting algorithm classifies inconsistently depending on intersection topology.
3. Very sparse shapes (< 10 points) are excluded from closure analysis due to unreliable metrics.
4. The 80% outline subtraction factor is empirically determined and may require adjustment for specific brush sizes or drawing styles.

### 8.4 Suggested Methodology Description for Publications

> Participants drew shapes on a 10 × 10 unit Cartesian grid (600 × 600 pixels; 60 pixels per unit) using circular brushes with configurable diameter (1–20 pixels, corresponding to 0.017–0.333 unit diameters). Shape areas were computed using a curve-aware pixel-based method with adaptive grid resolution (0.05–0.1 units) that measured the visual footprint of strokes by computing minimum distance to line segments between consecutive path points. For closed shapes, interior areas were determined via ray casting point-in-polygon testing, with 80% outline subtraction applied to prevent double-counting of stroke area.
>
> Shape closure was determined by a three-strategy classification system based on endpoint gap (as percentage of path length) and cumulative rotation (as percentage of 360°). Shapes meeting any of the following criteria were classified as closed: (1) gap < 2%; (2) gap < 10% and rotation ≥ 70%; or (3) gap < 15% and rotation ≥ 85%. Open shapes (failing all criteria) returned area = 0.
>
> Centroid coordinates were computed using the geometric median via Weiszfeld's iterative algorithm (convergence tolerance: 0.001 units), which guarantees placement within the drawn region. The algorithm automatically selected among four variants based on shape characteristics: basic medoid (default), skeleton-constrained (aspect ratio > 3.0), density-weighted (density variance > 0.5), or brush-aware (brush size > 15 pixels).

---

## 9. References

Weiszfeld, E. (1937). Sur le point pour lequel la somme des distances de n points donnés est minimum. *Tohoku Mathematical Journal*, 43, 355–386.

Preparata, F. P., & Shamos, M. I. (1985). *Computational Geometry: An Introduction*. Springer-Verlag. [Point-in-polygon algorithms, distance to line segment computation]

---

## 10. Version History

| Version | Date | Changes |
|---------|------|---------|
| 2.2 | November 2025 | Curve-aware distance calculation; three-strategy closure detection; explicit area = 0 for open shapes |
| 2.1 | November 2025 | Pixel-based area with interior filling; multi-criteria closure detection |
| 2.0 | November 2025 | Adaptive area and centroid calculation methods |
| 1.5 | November 2025 | Google Drive and Sheets integration |
| 1.0 | 2024 | Initial release |

---

**UCI Hearing & Speech Lab**  
**Sound Object Visualization Research Tool v2.2**
