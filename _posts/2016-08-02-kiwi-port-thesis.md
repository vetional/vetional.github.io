---
layout: post
title: "Porting Physics-Based Animations from C++ to Python"
comments: true
description: "Porting SPH fluid simulation from an IIT Madras thesis to Python and Kivy, and the performance tricks needed to make it interactive"
keywords: "sph, smoothed particle hydrodynamics, python, kivy, numpy, physics simulation, iit madras"
---

My M.S. thesis at IIT Madras was a Smoothed Particle Hydrodynamics (SPH) fluid simulator written in C++. It ran at 60fps for 10,000 particles. I wanted to port it to Python/Kivy for a cross-platform demo. The problem: naive Python is roughly 100x slower than C++ for this kind of tight numerical loop.

## Why SPH Is Computationally Expensive

SPH simulates fluids as particles. Each particle interacts with neighbors within a smoothing radius h. For every timestep, you compute density, pressure, and viscosity forces by summing contributions from all neighbors. With N particles, a brute-force approach is O(N^2).

The core computation looks like this in C++:

```cpp
// Density computation for particle i
for (int j = 0; j < num_neighbors; j++) {
    float r = distance(pos[i], pos[neighbors[j]]);
    if (r < h) {
        float q = r / h;
        density[i] += mass * kernel(q);  // cubic spline kernel
    }
}
```

Translating this directly to Python with for-loops gave about 2fps for 1000 particles. Completely unusable.

## NumPy Vectorization

The key insight: replace per-particle Python loops with bulk NumPy operations. Instead of iterating over particles, compute all pairwise distances as matrix operations.

```python
import numpy as np

def compute_density(positions, mass, h, neighbor_lists):
    N = len(positions)
    density = np.zeros(N)

    for i in range(N):
        nbrs = neighbor_lists[i]
        if len(nbrs) == 0:
            continue
        diff = positions[nbrs] - positions[i]  # vectorized subtraction
        r = np.linalg.norm(diff, axis=1)
        mask = r < h
        q = r[mask] / h
        # Cubic spline kernel, vectorized
        w = (1.0 - 1.5*q**2 + 0.75*q**3)
        w[q > 0.5] = 0.25 * (2.0 - q[q > 0.5])**3
        density[i] = mass * np.sum(w)

    return density
```

This still has a Python loop over particles, but the inner computation is vectorized. For 1000 particles with ~30 neighbors each, this ran at 18fps. Better, but not interactive.

## Spatial Hashing for Neighbor Search

The O(N^2) neighbor search was the real bottleneck. We used a spatial hash grid: divide space into cells of size h, and only check particles in adjacent cells.

```python
def build_spatial_hash(positions, h):
    grid = {}
    cell_size = h
    for i, pos in enumerate(positions):
        key = (int(pos[0] / cell_size), int(pos[1] / cell_size))
        grid.setdefault(key, []).append(i)
    return grid

def find_neighbors(grid, positions, i, h):
    cell_size = h
    cx = int(positions[i][0] / cell_size)
    cy = int(positions[i][1] / cell_size)
    neighbors = []
    for dx in (-1, 0, 1):
        for dy in (-1, 0, 1):
            key = (cx + dx, cy + dy)
            if key in grid:
                for j in grid[key]:
                    if i != j:
                        r = np.linalg.norm(positions[i] - positions[j])
                        if r < h:
                            neighbors.append(j)
    return neighbors
```

With spatial hashing, neighbor search dropped from O(N^2) to roughly O(N). Combined with vectorized force computation, we hit 30fps for 1000 particles on a laptop. Good enough for an interactive demo.

## When to Use Python vs C

For 1000 particles, Python with NumPy was adequate. For 10,000+ particles, you need C/C++ or at minimum Cython for the inner loops. The thesis code handled 50,000 particles in C++ at 30fps. Python topped out at about 2000 particles for interactive rates.

The Kivy rendering was not the bottleneck. Drawing 1000 circles as Kivy widgets was fast. The physics computation dominated. If I were doing this again, I would write the simulation in C as a shared library and call it from Python via ctypes, keeping Kivy only for rendering.

## What Worked and What Didn't

NumPy vectorization gave a 9x speedup over naive Python loops. Spatial hashing gave another 4x. Together they made 1000-particle simulations interactive. The Kivy framework was pleasant to work with for 2D rendering and handled touch input well on both desktop and mobile.

What didn't work: I tried using Python multiprocessing to parallelize the force computation. The overhead of serializing NumPy arrays between processes was larger than the computation itself for 1000 particles. Parallelism only helps at larger scales where the per-particle work dominates the communication cost.
