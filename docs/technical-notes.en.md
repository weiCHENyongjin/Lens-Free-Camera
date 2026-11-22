# Technical Notes: Optical Encoding and 3D Inversion

[简体中文](technical-notes.zh-CN.md) · [Project home](../README.en.md)

## 1. Coordinates and discrete scene

Lateral coordinates `x,y` are defined on the diffuser plane. Depth `z` is the positive point-source-to-diffuser distance. The demo samples `x∈[-55,55] mm` and `y∈[-45,45] mm` on a `20×16` grid, at `z={0.75,0.95,1.10,1.55,1.85,2.20} m`. Volume arrays consistently use `(z,y,x)` axis order.

The scene builder applies a front-view visibility rule and keeps the nearest visible surface at each `(x,y)` location. It illustrates occlusion for opaque scenes. It is not a general mesh rasterizer and does not synthesize shadows or inter-reflections.

## 2. Random phase screen and propagation

Default sampling and optical parameters:

| Quantity | Value |
| --- | ---: |
| Wavelength `λ` | 532 nm |
| Sampling grid | 96×96 |
| Sensor pixel pitch | 8 μm |
| Sampled sensor width | 0.768 mm |
| Diffuser-to-sensor distance `L` | 3 mm |
| Phase correlation length | 2 px |
| Phase standard deviation | 2.5 rad |
| Angular-spectrum zero-padding factor | 2 |

The phase is generated from white noise with a Gaussian frequency-domain filter, then scaled to the requested standard deviation. The transmission is `T(x,y)=exp(iφ(x,y))`. The same phase screen is reused for calibration and capture. Changing the seed changes the phase texture and the PSF dictionary.

For a point source at `(x_s,y_s,z)`, the incident spherical wave is approximated by:

```text
r(x,y) = sqrt((x-x_s)^2 + (y-y_s)^2 + z^2)
U_d(x,y) = A exp(i k r) / r,  k = 2π/λ
Σ_pixels |U_d|² = 1
```

After multiplication by the diffuser, propagation to the sensor uses the angular-spectrum transfer function:

```text
H(f_x,f_y) = exp(i k L sqrt(1 - (λf_x)² - (λf_y)²))
U_s = F⁻¹{F{U_d T} H}
h_j = |U_s|²
```

Only propagating frequencies are retained; evanescent frequencies are set to zero. The complex field is zero-padded before the FFT and cropped to the finite sensor afterward. PSFs are not renormalized after cropping, so their sums report captured energy. In the seed-7 demo, captured fractions range from `0.794` to `0.944`, with median `0.879`.

## 3. Incoherent measurement and dead-pixel mask

For `N` candidate voxels, let `x_j≥0` be brightness and `h_j` the PSF for voxel `j`. The noiseless sensor image is:

```text
y = Σ(j=1..N) x_j h_j
```

This is incoherent intensity addition, not coherent complex-amplitude addition. The simulator optionally samples Poisson photon counts and adds Gaussian read noise:

```text
y_photon = Poisson(photon_count × y) / photon_count
y_noisy = y_photon + Normal(0, noise_std²)
```

The demo uses `photon_count=500000`. `noise_std=0` means no additional Gaussian term; Poisson noise remains enabled.

A pixel is excluded if it is missing from the measurement or any relevant calibration PSF. Invalid entries are stored as NaN and handled through a common validity mask. The 96×96 demo retains 8,136 valid samples. A missing pixel is not a zero-intensity observation.

## 4. Non-negative L1 inversion and spatial smoothness

Each valid PSF is flattened into a column of dictionary matrix `D`, then internally normalized by its L2 norm. The base objective is:

```text
minimize_x  0.5 ||D x - y||²₂ + λ ||x||₁
subject to  x ≥ 0
```

`λ = relative_l1 × max(Dᵀy)`, so `relative_l1` scales with the largest dictionary correlation for the current measurement. The default is `0.003`. FISTA estimates its step-size bound by power iteration and stops at the iteration limit or when relative coefficient change falls below tolerance. Before returning, the solver reverses PSF-column normalization to restore brightness scale.

An optional smoothness term penalizes lateral neighbors independently within each depth slice:

```text
objective += (μ/2) Σ_z Σ_(neighbor pairs in x,y) (x_a - x_b)²
```

The default is `μ=0.03`. It neither smooths across depth nor encodes object geometry or surface normals. In single-surface mode, the solver first reconstructs the full volume, keeps the strongest depth on each ray, and refits brightness using only the selected PSFs.

## 5. Residual-gated depth coordinate updates

Depth refinement runs only on a single-surface support. It first computes relative residual on valid pixels:

```text
ρ = ||y_fit - y||₂ / max(||y||₂, ε)
```

If `ρ > depth_refinement_residual_threshold`, the single-surface refit is returned unchanged. The default threshold is `0.05`. This gate avoids interpreting random noise as depth evidence when the initial fit is poor.

When `ρ` is within threshold, each active `(x,y)` ray is updated in turn:

1. Remove that ray's current PSF contribution from the sensor residual.
2. Score every calibrated depth at the same lateral position using the non-negative L1 update.
3. Select the highest-scoring depth and solve its brightness analytically. Keep the current depth for near-ties.
4. Sweep in a deterministic pseudo-random order seeded with `42`, for up to `depth_refinement_steps` rounds. Stop early when neither coefficients nor depths change.

This is greedy coordinate optimization. It depends on the initial support and is not guaranteed to find a global optimum. It does not activate new lateral positions outside the initial support, and it is not appropriate for transparent objects or multiple true surfaces on the same ray. The API leaves this step off by default; enabling it requires `single_surface=True`.

## 6. Public NPZ data dictionary

[`meeting_scene_results.npz`](../data/meeting_scene_results.npz) is a compressed NumPy NPZ archive. Main arrays:

| Key | Shape | Meaning |
| --- | --- | --- |
| `phase_screen_rad` | `(96,96)` | Fixed phase screen in radians |
| `calibration_psfs_partial` | `(1920,96,96)` | Per-voxel PSFs with missing samples set to NaN |
| `sensor_mask` | `(96,96)` | Boolean valid-pixel mask |
| `measurement` | `(96,96)` | Sensor capture, with missing entries set to NaN |
| `true_volume` | `(6,16,20)` | Synthetic ground-truth brightness volume |
| `reconstructed_volume` | `(6,16,20)` | Reconstructed brightness volume |
| `source_positions_xyz_m` | `(1920,3)` | `(x,y,z)` position for each PSF, in meters |
| `depths_m`, `x_positions_m`, `y_positions_m` | `(6)`, `(20)`, `(16)` | Coordinates on each voxel-grid axis |

The archive also records `spatial_smoothness`, `relative_l1`, `single_surface`, `sensor_size`, `depth_refinement_steps`, and the residual threshold. Brightness values are simulation coefficients, not physical reflectance or absolute optical flux.
