# Changelog

This changelog starts at version 1.4.0. For earlier versions see the
[commit history](https://github.com/ladisk/pyidi/commits/master).

## Unreleased

### New method: SimplifiedOpticalFlowPyramid

`SimplifiedOpticalFlowPyramid` runs the simplified optical flow on a
coarse-to-fine image pyramid. Each point is handled on its own: the method
averages the reference intensity and both image gradients over a subset around
the point, reads the displacement along the gradient direction from the
intensity difference, and projects it onto the image axes through the gradient
components.

Plain simplified optical flow only resolves displacements up to about half a
pixel. On the pyramid, the estimate from a coarser level is scaled up and
applied as a shift of the subset on the next level, rounded to whole pixels
by default, so only the
sub-pixel residual is measured at full resolution. On a synthetic stripe
pattern translated by 6.7 px, a single level is off by more than 28 px, while
four levels stay within 0.2 px.

`configure(interpolation='linear')` and `configure(interpolation='cubic')`
keep the shift from a coarser level fractional and read the subset with a
spline of order 1 or 3, instead of rounding the shift to whole pixels. Both
remove the step the rounding leaves in the result. The linear spline smooths
the image enough to bias the result below about one pixel of displacement,
where rounding shifts by nothing at all; the cubic spline does not. Measured
on a speckle pattern in circular motion, as the error along the gradient in
px:

| largest displacement | `'nearest'` | `'linear'` | `'cubic'` |
| --- | --- | --- | --- |
| 0.2 px | 0.005 | 0.029 | 0.002 |
| 0.5 px | 0.036 | 0.055 | 0.005 |
| 1 px | 0.048 | 0.060 | 0.017 |
| 2 px | 0.095 | 0.086 | 0.077 |

The subset can be rectangular: `configure(subset_size=(size_y, size_x))` takes
any pair of odd sizes, and a single int still means a square subset. Choose the
levels either with `n_levels`, which uses levels 0 to `n_levels - 1`, or with
`levels`, which also lets you skip levels, as in `levels=[2, 0]`.

The subset means are read from an integral image (summed-area table), so each
point costs four reads whatever its subset size. A subset that reaches over the
image edge is averaged over the pixels inside the image only.

On a coarse level a point keeps its fractional position, `point / scale**level`,
and its subset is read with interpolation instead of being moved to the nearest
pixel of that level. Rounding put the coarse subset up to half a pixel of the
level off the point, which on a line edge moved by 4 px left an error of
0.52 px; without rounding the error is 0.03 px.

## 1.4.0

### Lucas-Kanade performance

The inner optimization loop is now compiled with `numba` and parallelized over
points. Measured against 1.3.3 on the same machine, with identical results:

| case | 1.3.3 | 1.4.0 | speedup |
| --- | --- | --- | --- |
| `data_synthetic.cih`, 200 points, 101 frames | 7.94 s | 0.10 s | 77x |
| synthetic 512x512, 400 points, 150 frames | 21.04 s | 0.24 s | 89x |
| `data_synthetic.mp4`, 60 points, 10 frames | 2.16 s | 0.06 s | 36x |

The compiled path is on by default. `configure(use_compiled_kernel=False)`
selects the previous NumPy implementation, which is itself faster than 1.3.3
(about 1.6x, and more on video and image files) because the frame is no longer
re-read for every point. Results from the two paths agree to floating-point
round-off. If numba cannot be imported, pyidi falls back to the NumPy
implementation automatically and warns, rather than failing to import.

The compiled kernel supports cubic interpolation only; `int_order` other than `3`
falls back to the NumPy implementation and warns once.

### Directional Lucas-Kanade performance

`DirectionalLucasKanade` now uses the same compiled kernel machinery. The spline
evaluation is shared with Lucas-Kanade; only the least-squares solve differs,
because the motion is constrained to one prescribed direction and so has a
single unknown instead of two. Measured against 1.3.3 on the same machine, with
identical results:

| case | 1.3.3 | 1.4.0 | speedup |
| --- | --- | --- | --- |
| `data_synthetic.cih`, 120 points, 101 frames | 1.46 s | 0.04 s | 38x |
| synthetic 512x512, 400 points, 150 frames | 7.46 s | 0.11 s | 65x |
| `data_synthetic.mp4`, 60 points, 10 frames | 1.25 s | 0.03 s | 41x |

As for Lucas-Kanade, the frame is now read once per time step instead of once
per point. That alone makes the NumPy path about 18x faster on video and image
files; on memory-mapped formats it makes no measurable difference.

### Behaviour changes

- **A point that cannot be tracked no longer aborts the analysis.** Previously a
  degenerate region of interest raised `ValueError` and no results were returned
  at all. Such a point is now `NaN` from the frame at which it was lost, every
  other point is computed normally, and a warning is issued. Which points failed
  is recorded in the new `failed_points` attribute. This applies to
  `use_compiled_kernel=False` as well, so code that relied on the exception to
  halt should check for `NaN` instead. It applies to both `LucasKanade` and
  `DirectionalLucasKanade`.
- The NumPy path now also detects a runaway optimization. Previously such a point
  returned displacements of tens of pixels, silently.
- `DirectionalLucasKanade.configure()` no longer accepts `use_numba`. The
  argument was accepted but documented as not implemented and did nothing; it is
  replaced by `use_compiled_kernel`, which defaults to `True` and does what the
  old name suggested. Calls that passed `use_numba` need updating.
- `DirectionalLucasKanade` previously warned and continued with a zero update
  when the image had no gradient along the search direction, which returned a
  point that silently never moved. Such a point is now reported through
  `failed_points` and set to `NaN`.
- `DirectionalLucasKanade.configure()` accepts a scalar `pad` in addition to the
  `(pad_y, pad_x)` pair it required before.

Detection of untrackable points is best effort: a point can return implausible
values without being flagged, so a result without `NaN` is not proof that every
point tracked correctly.

### Other

- `numba` now requires at least 0.59.
- Windows and macOS added to the CI test matrix.
- When `processes` is greater than one, each worker is limited to a single numba
  thread, so process-level and thread-level parallelism cannot oversubscribe the
  CPU. This applies to both Lucas-Kanade methods.
- pyidi now asks numba for a fork-safe threading layer as the first thing it
  does, before importing anything that imports numba. This matters because
  pyMRAW starts numba's thread pool at import time, and the layer cannot be
  changed once the pool is up. Left alone it resolves to whatever is installed:
  TBB on machines that have it, but OpenMP on machines that do not, and GNU
  OpenMP kills any child forked from a process that has used it. Set
  `NUMBA_THREADING_LAYER` to override.
- On Linux, the worker pool no longer forks when a GNU OpenMP runtime is loaded
  into the process, since libgomp terminates any child forked from a process
  that has used it. numba's own threading layer is fork-safe, but OpenCV and
  some BLAS and SciPy builds pull in libgomp too, and that was enough to kill
  every worker. Such sessions now start their workers through `forkserver`.
  Where forking is safe it is still used, because it shares the video with the
  workers rather than pickling a copy to each of them.
- The kernels are compiled once in the parent process before the worker pool is
  created, instead of once per worker.
- `DirectionalLucasKanade`'s `compute_delta_numba` is now actually compiled; its
  `@numba.njit` decorator had been commented out. The name stays importable as an
  alias of the new `compute_delta`.
