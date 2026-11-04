<div align="center">

# Design choices

[![Design choices 14](https://img.shields.io/badge/Design%20choices-14-6F42C1)](#d1-reference-spectrum-for-the-offset-correction)
[![Pipeline steps 3-11](https://img.shields.io/badge/Pipeline%20steps-3--11-E8833A)](README.md#pipeline)
[![README](https://img.shields.io/badge/Back%20to-README-24292F)](README.md)

</div>

These are the parameter and method choices in the pipeline, with the reasons behind them. Each
entry gives the pipeline step, where the choice sits in the code, the value used, and why it was
chosen. Step numbers follow the pipeline flowchart in the README.

---

## D1. Reference spectrum for the offset correction

![Calibration and preprocessing](https://img.shields.io/badge/Calibration%20and%20preprocessing-1f6feb) ![Step 3](https://img.shields.io/badge/Step-3-555555)

- **Step:** 3. Offset correction
- **Code:** `raw_to_temperature.ipynb`, `crop_window = 8`
- **Choice:** the reference spectrum is the sum of the central 8 x 8 scan pixels.

> **Justification:** The scan centre is taken as the zero-offset reference. Summing 8 x 8 pixels averages over many spectra, so the reference is less noisy than any single pixel.

---

## D2. Energy calibration from PINEM sidebands

![Calibration and preprocessing](https://img.shields.io/badge/Calibration%20and%20preprocessing-1f6feb) ![Step 4](https://img.shields.io/badge/Step-4-555555)

- **Step:** 4. Energy axis calibration
- **Code:** `fit_pinem_calib`
- **Choice:** one global dispersion and energy offset from the first-order PINEM sidebands (±ħω),
  each fitted with a Gaussian and a linear background.

> **Justification:** Calibration only needs the peak centres, not an exact line shape. A Gaussian finds the centre to better than one energy channel, so a Lorentzian was not needed.

---

## D3. Laser pulse edge detection

![Calibration and preprocessing](https://img.shields.io/badge/Calibration%20and%20preprocessing-1f6feb) ![Step 5](https://img.shields.io/badge/Step-5-555555)

- **Step:** 5. Pulse detection using PINEM
- **Code:** `PINEM_time`, `poly_order = 12`
- **Choice:** the PINEM trace of the -ħω (energy-gain) sideband, smoothed with a 12th-order
  polynomial. `t_on` and `t_off` are the first and last zero crossings of its second derivative.
  The alternative, the maximum and minimum of the first derivative, is left commented out in the code.

> **Justification:** The zero crossings of the second derivative mark the inflection points, where the PINEM intensity starts to rise and starts to fall. The method is preliminary, but it is automated and works better than the earlier approach.

---

## D4. Time window for the bulk plasmon pre-fit

![Temperature map extraction](https://img.shields.io/badge/Temperature%20map%20extraction-e8833a) ![Step pre-fit](https://img.shields.io/badge/Step-pre--fit-555555)

- **Step:** bulk plasmon pre-fit (seeds step 9)
- **Code:** `fit_bulk_plasmon`
- **Choice:** from `t_on - 0.1 * t_on` to `t_off + 0.8 * t_off`, about 255 ns before the pulse to
  2 µs after it for scan 02.

> **Justification:** The window covers a short time before the pulse and a much longer time after it, so the fit also sees the cooling dynamics.

---

## D5. Crop window

![Temperature map extraction](https://img.shields.io/badge/Temperature%20map%20extraction-e8833a) ![Step 6](https://img.shields.io/badge/Step-6-555555)

- **Step:** 6. Crop and rebin
- **Code:** `time_start_indx`, `time_end_indx`, `energy_start_indx`, `energy_end_indx`
- **Choice:**
  - time: start at `0.8 * t_on`, with 2^13 + 2^11 + 2^10 = 11264 channels;
  - energy: start at 12 eV, with 2^8 = 256 channels.

> **Justification:** Window lengths are sums of powers of 2, so they divide evenly by the rebin factors. In time, the window is made as long as possible after `t_on`. In energy, it is cropped to the region around the bulk plasmon.

---

## D6. Rebin factors

![Temperature map extraction](https://img.shields.io/badge/Temperature%20map%20extraction-e8833a) ![Step 6](https://img.shields.io/badge/Step-6-555555)

- **Step:** 6. Crop and rebin
- **Code:** `ce_crop_hyperspim.rebin(...)`
- **Choice:** t x8, sx x8, sy x8, E x2 (scan 02: 256 x 256 pixels become 32 x 32).

> **Justification:** Rebinning trades spatial, temporal and energy resolution against noise. Several combinations were tested, and these factors gave the best balance for the final data set.

---

## D7. Number of PCA components

![Temperature map extraction](https://img.shields.io/badge/Temperature%20map%20extraction-e8833a) ![Step 7](https://img.shields.io/badge/Step-7-555555)

- **Step:** 7. Build te_hyperspim object and PCA
- **Code:** `masked_PCA_hyperspim`, `get_decomposition_model(10)`
- **Choice:** reconstruct from the first 10 components.

> **Justification:** The explained variance of each component was checked (`plot_explained_variance_ratio`), and the components carrying most of the variance were kept. This was checked for each reconstruction during the preliminary analysis.

---

## D8. Aluminium mask threshold

![Temperature map extraction](https://img.shields.io/badge/Temperature%20map%20extraction-e8833a) ![Step 8](https://img.shields.io/badge/Step-8-555555)

- **Step:** 8. Aluminium mask
- **Code:** `masked_PCA_hyperspim`, `threshold_factor = 0.25`
- **Choice:** a pixel is kept if the maximum of its time-summed spectrum is above 25 % of the global
  maximum.

> **Justification:** The threshold was checked visually against the HAADF image, so that only the aluminium region remains. If too much of the aluminium is masked out, the threshold is lowered slightly.

---

## D9. Per-pixel fit model and bounds

![Temperature map extraction](https://img.shields.io/badge/Temperature%20map%20extraction-e8833a) ![Step 9](https://img.shields.io/badge/Step-9-555555)

- **Step:** 9. Multifit: per-pixel fitting
- **Code:** `masked_multifit_PCA_hyperspim`
- **Choice:** a Lorentzian and a linear background. The centre is bounded to ±20 % and the width
  (gamma) to ±80 % of the starting values from the pre-fit.

> **Justification:** The per-pixel plasmon centre is not expected to move far from the summed bulk plasmon response, so the summed fit is used both as the starting value and to set the bounds of each fit.

---

## D10. Reference plasmon energy E_p(T0)

![Temperature map extraction](https://img.shields.io/badge/Temperature%20map%20extraction-e8833a) ![Step 10](https://img.shields.io/badge/Step-10-555555)

- **Step:** 10. Energy to temperature
- **Code:** `hyperSPIM_temp_plot`, `pre_laser_centres = hyperspim_centre_hs.data[:,:,0:90]`
- **Choice:** the per-pixel mean of the first 90 frames, which are before the pulse, plus the
  largest per-pixel standard deviation.

> **Justification:** The reference is the bulk plasmon energy before heating starts. It stands for the plasmon energy at ambient temperature.

---

## D11. Root selection and outlier handling

![Temperature map extraction](https://img.shields.io/badge/Temperature%20map%20extraction-e8833a) ![Step 10](https://img.shields.io/badge/Step-10-555555)

- **Step:** 10. Energy to temperature
- **Code:** `delta_E_to_T_array`, `max_T = 1000`, `max_iter = 50`
- **Choice:**
  - α(T) is quadratic, so the conversion is a cubic in ΔT;
  - the smallest positive real root is kept;
  - roots above `max_T` count as outliers: ΔE_p is changed in steps of ±0.5 % for up to 50 steps,
    and if no root is found, ΔT = 0.

> **Justification:** Only one of the three roots of the cubic is physical. One root is negative, and another is above 4000, far beyond the melting point of aluminium (660 °C), so both are rejected. If no valid root exists, the measured ΔE_p has been distorted by instrumental error. It is then changed in small steps until a valid temperature is found. The steps are small enough that the result stays within the acceptable error range.

---

## D12. Fit model

![Temperature map extraction](https://img.shields.io/badge/Temperature%20map%20extraction-e8833a) ![Step 11](https://img.shields.io/badge/Step-11-555555)

- **Step:** 11. Temperature model fitting
- **Code:** `temperature_fitting.ipynb`, `fit_rise = True`, `fit_model = 2`, `model2_fn`
- **Choice:** rise and decay are fitted together with separate amplitudes,
  `A_1 * exp(-dt/tau_decay) - A_2 * exp(-dt/tau_rise) + C`, rather than one shared amplitude
  (model 1) or a decay-only fit.

> **Justification:** The rise and the decay have different physical origins: the rise is driven by the laser actively pumping heat in, while the cooling follows heat-transport laws. A single shared amplitude ties the two together. With separate amplitudes, the peak region is fitted better and the fits are more reliable, so the decoupled model was used.

---

## D13. Initial guesses and bounds of the transient fit

![Temperature map extraction](https://img.shields.io/badge/Temperature%20map%20extraction-e8833a) ![Step 11](https://img.shields.io/badge/Step-11-555555)

- **Step:** 11. Temperature model fitting
- **Code:** `fit_pixel`, `tau_rise_init = 15.0`, `tau_decay_init = 300.0`
- **Choice:**
  - baseline C from the mean of the first 20 frames; A_1 from the maximum above C; A_2 = 0.5 * A_1;
  - bounds: amplitudes >= 0, tau_rise 0.5-200 ns, tau_decay 5-5000 ns.

> **Justification:** C is the baseline the rise starts from. It is visible in the first few frames, before the pulse, so their mean is a good starting value. The other bounds keep the parameters within physically reasonable values.

---

## D14. Background mask and display limits of the exported maps

![Temperature map extraction](https://img.shields.io/badge/Temperature%20map%20extraction-e8833a) ![Step 11](https://img.shields.io/badge/Step-11-555555)

- **Step:** 11. Temperature model fitting
- **Code:** `bridge_mask`, `plot_config`, `make_correction_map`
- **Choice:**
  - pixels whose ΔT trace is constant (all zero, outside the aluminium mask) are drawn in grey;
  - colour limits are set per parameter at about the 2nd-98th percentile of the bridge pixels;
    `chi2_red` uses a log scale.

> **Justification:** Drawing the non-aluminium pixels in grey keeps the final maps clean, with only the bridge and pads in colour. The colour limits ignore outliers, so that the contrast within the bridge and the pads stays visible.
