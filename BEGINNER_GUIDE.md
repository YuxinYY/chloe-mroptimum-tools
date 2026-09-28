# MR Optimum Tools: a beginner's guide

This repository calculates **signal-to-noise ratio (SNR)** maps from MRI raw
data.  It is a research/development tool: it reads the measurements acquired
by an MRI scanner, reconstructs an image, estimates its SNR, and writes image
files that can be inspected in an MRI image viewer.  It is not a clinical
diagnosis tool.

You do **not** need an MRI scanner or a patient dataset to try the project.
The fastest first run creates a synthetic image, simulates an MRI receive-coil
array and noise, and uses those data to exercise the core SNR calculation.

## The MRI vocabulary used here

MRI scanners do not first save a normal picture.  Each receiver coil records
complex-valued frequency-domain measurements called **k-space**.  A Fourier
transform turns k-space into an image.  Several receiver coils observe the
same anatomy from different positions; combining them can improve image
quality, but it also means that noise and coil sensitivity have to be handled
carefully.

| Term | Plain-language meaning | Why this project needs it |
| --- | --- | --- |
| **SNR** | Signal strength relative to random measurement variation (noise). Higher usually means a clearer measurement. | The primary output is an SNR value at every image location (a map). |
| **k-space** | The scanner's complex raw measurements, before they are converted to a conventional image. | This is the main input: Siemens `.dat`, NumPy `.npy`/`.npz`, or MATLAB `.mat`. |
| **coil** | One antenna in the array used to receive MRI signal. | The final k-space axis represents coils; each coil has a different view and noise behavior. |
| **noise scan** | Data acquired without useful anatomy signal, used to measure noise level and correlation across coils. | Enables a noise covariance estimate, which is important for meaningful SNR. |
| **reconstruction** | The method that turns multi-coil k-space into an image. | Choose RSS, B1, SENSE, or GRAPPA in the JSON configuration. |
| **acceleration** | Deliberately skipping some k-space measurements to scan faster. | SENSE and GRAPPA use it, along with reference (ACS) data, to fill in missing information. |
| **ACS/reference data** | Fully sampled central k-space used to calibrate accelerated reconstruction. | Needed by SENSE/GRAPPA when it cannot be obtained from the signal file. |
| **flip angle (FA)** | How far the scanner's radiofrequency pulse tips magnetization, measured in degrees. | The optional FA correction reports a result normalized to a 90-degree flip angle. |
| **NIfTI** | A common file format for 3-D medical images, usually named `.nii.gz`. | Command-line results are written in this format so they retain voxel geometry. |

SNR is not a universal property of a person or an image.  It depends on the
acquisition, coils, reconstruction, and the SNR-estimation method.  When
comparing two SNR maps, use the same method and configuration unless you are
deliberately studying the difference.

## What is in this repository

```text
mrotools/
  snr.py                 command-line SNR pipeline
  mro.py                 connects SNR methods to reconstruction classes
  kspace_loaders.py      reads Siemens, NumPy, and MATLAB k-space
  fa_normalization.py    optional SNR / sin(flip angle) correction
  collections/           example JSON configurations
tests/
  test_numpy_snr.py      self-contained synthetic end-to-end smoke test
  test_fa_normalization.py   focused tests of FA correction
  test_dat_version.py    tests Siemens version/header interpretation
tools/                   converters and inventory helpers for raw data
README.md                detailed implementation and method reference
CONTRIBUTING.md          developer-oriented testing notes
```

The package relies on `cmtools`, `pynico_eros_montin`, and
`pyable_eros_montin` for much of the reconstruction, image I/O, and support
code.  Their required versions are declared in `pyproject.toml`.

## Set up a local environment

From the repository root, create an isolated environment.  Python 3.11 is a
good starting point because the repository's developer instructions use it
(the package declares support for Python 3.9 and later).

```bash
python3.11 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
python -m pip install -e . pytest
```

The editable install (`-e .`) means edits to `mrotools/` are used immediately.
Installation downloads a few dependencies from their upstream Git repositories,
so it needs network access.  If your team uses Conda, the equivalent setup is:

```bash
conda create -n mrotools-test python=3.11 -y
conda activate mrotools-test
python -m pip install -e . pytest
```

## First successful run: no MRI data required

Run the synthetic end-to-end example and keep its results in a local folder:

```bash
python tests/test_numpy_snr.py --output ./example-output
```

The script creates a simple phantom (an artificial image), simulates eight
coils and complex noise, writes `signal.npy`, `noise.npy`, and `config.json`,
then calculates an analytical RSS SNR map.  A successful run ends by reporting
`example-output/SNR.nii.gz`.

This is a runnable **smoke test**, not a pytest test module: execute it with
Python as shown above.  You can experiment without changing code:

```bash
python tests/test_numpy_snr.py --output ./example-output --coils 4 --noise 0.03 --size 96 96
```

More coils and lower noise generally raise the simulated SNR, but the exact
numbers are not a real scanner validation target.

## Running the command-line pipeline

The command-line entry point accepts a JSON configuration and an output
directory:

```bash
python -m mrotools.snr \
  --joptions /absolute/path/to/config.json \
  --output /absolute/path/to/output-directory \
  --no-parallel
```

`--no-parallel` is convenient for a first run and debugging.  Remove it to
process multiple slices in parallel.  Useful optional flags are:

| Flag | Effect |
| --- | --- |
| `--coilsens` | Save estimated coil-sensitivity maps when the reconstruction has them. |
| `--gfactor` | Save the g-factor for SENSE/GRAPPA, a map of noise amplification caused by acceleration. |
| `--matlab` | Also save MATLAB-format output. |
| `--verbose` | Display plots; use only in an interactive graphical session. |
| `--fa-map /path/to/fa.nii.gz` | Apply optional flip-angle normalization. FA values must be in degrees. |
| `--fa-interpolation bspline` | Choose FA resampling (`bspline` is the default; `nearest` and `linear` are also available). |

The normal output folder contains `info.json` plus a `data/` directory.  The
main map is normally `data/SNR.nii.gz`.  If noise data were supplied, it also
contains noise covariance outputs.  With `--fa-map`, the code additionally
writes `data/SNR_90deg.nii.gz` and `data/FA_on_SNR.nii.gz`.  `info.json`
records the configuration, output inventory, timing, and log entries.

Open `.nii.gz` images with a medical-image viewer such as 3D Slicer, ITK-SNAP,
or FSLeyes.  For SNR images, inspect the magnitude/absolute-value display;
some pipeline outputs are complex-valued internally.

## Your first configuration

The configuration tells the tool which SNR estimator and reconstruction to
use, and where to find k-space.  Start with analytical SNR (`AC`) and root-sum
of-squares (`RSS`), the least demanding combination.  Save the following as
`config.json`, substituting absolute paths to your k-space files:

```json
{
  "version": "v0",
  "acquisition": 2,
  "type": "SNR",
  "name": "AC",
  "options": {
    "reconstructor": {
      "type": "recon",
      "name": "RSS",
      "options": {
        "signal": {
          "type": "file",
          "options": {
            "type": "local",
            "vendor": "numpy",
            "filename": "/absolute/path/to/signal.npy"
          }
        },
        "noise": {
          "type": "file",
          "options": {
            "type": "local",
            "vendor": "numpy",
            "filename": "/absolute/path/to/noise.npy"
          }
        }
      }
    }
  }
}
```

Then run the command in the previous section.  The committed templates in
`mrotools/collections/numpy/` and `mrotools/collections/matlab/` show more
options, but update their placeholder or machine-specific file paths before
using them.

### Input data requirements

For a single 2-D slice, NumPy or MATLAB k-space must be a **complex** array
with shape:

```text
(frequency samples, phase samples, coils)
```

For several 2-D slices, use:

```text
(frequency samples, phase samples, coils, slices)
```

The noise input should have compatible coil count and be complex k-space as
well.  A bare `.npy` works.  A `.npz` can additionally carry geometry and
acceleration metadata:

```python
np.savez(
    "signal.npz",
    kspace=kspace,                         # required complex array
    spacing=[1.0, 1.0, 5.0],               # mm per voxel
    origin=[0.0, 0.0, 0.0],                # image-coordinate starting point
    direction=[1, 0, 0, 0, 1, 0, 0, 0, 1], # 3 x 3 orientation, row-major
    acceleration=[1, 2],                   # optional: frequency, phase
    acl=[0, 24],                            # optional calibration-line count
)
```

Geometry is resolved in this order: an `orientation` block in the JSON,
metadata embedded in `.npz`/`.mat`, then defaults of 1 mm spacing, zero origin,
and identity direction.  Those defaults make a calculation runnable but may
not represent the real physical geometry; provide real metadata whenever the
output will be aligned with other medical images.

Supported values for `vendor` are `numpy`, `matlab`, and `siemens`.  Siemens
uses raw `.dat` data via `twixtools`; for a portable workflow, use
`tools/dat2numpy.py` to convert an accessible Siemens scan into self-contained
`.npz` files plus a starter configuration.  `tools/ismrmrd2numpy.py` performs
the analogous conversion for ISMRMRD `.h5` input (and needs the optional
`ismrmrd` package).

## Choosing an SNR method and reconstruction

Choose the simplest option that matches how your data were acquired.  The
method names below are the values placed in JSON, not judgments about image
quality.

| JSON SNR name | Meaning | Use it when |
| --- | --- | --- |
| `AC` | Analytical (Kellman-style) estimate | You have a signal/noise acquisition and need a direct estimate. A good first choice. |
| `MR` | Multiple Replicas | You have multiple independently repeated image acquisitions. |
| `PMR` | Pseudo Multiple Replicas | You want a simulation-based replica estimate; configuration includes `NR`, the number of replicas. |
| `CR` | Coil Replica / generalized PMR | You need this specialized local estimate; configuration includes `NR` and `boxSize`. |

| JSON reconstruction name | Meaning | Extra requirements |
| --- | --- | --- |
| `RSS` | Root-sum-of-squares coil combination | No sensitivity map or acceleration information. Best first run. |
| `B1` | Sensitivity-weighted coil combination | Coil-sensitivity estimation/reference data. |
| `SENSE` | Sensitivity encoding for accelerated scans | Acceleration, calibration/reference data, and sensitivity information. |
| `GRAPPA` | k-space interpolation for accelerated scans | Acceleration and ACS/reference data; optionally tune `kernelSize`. |

For SENSE or GRAPPA, the tool checks headers or embedded metadata for
`acceleration` and calibration-line (`acl`) information.  You can explicitly
set `accelerations` and `acl` in the reconstruction options.  If there is no
real acceleration, do not choose SENSE/GRAPPA just to make an image: RSS or B1
is the appropriate starting point.

## Optional flip-angle normalization

If you have a flip-angle map in degrees, this command estimates an SNR map
normalized to 90 degrees:

```bash
python -m mrotools.snr \
  -j /absolute/path/to/config.json \
  -o /absolute/path/to/output-directory \
  --fa-map /absolute/path/to/fa-map.nii.gz \
  --fa-interpolation bspline
```

Conceptually, it divides SNR by `sin(FA)`.  The FA map is resampled to the SNR
image geometry when necessary.  Values near 0 degrees are not physically safe
to divide by, so those pixels are masked rather than producing infinite SNR.
Review `FA_on_SNR.nii.gz` before interpreting the corrected map.  See the
FA-normalization section of `README.md` for Siemens DICOM scaling details.

## How to test changes

After installing the package and pytest in the environment, run the focused
tests that need no scanner data:

```bash
python -m pytest tests/test_fa_normalization.py tests/test_dat_version.py -v
python tests/test_numpy_snr.py --output ./example-output
```

The first command checks flip-angle math/resampling and Siemens header-version
parsing.  The second performs the synthetic end-to-end SNR calculation.

To try every example configuration, use:

```bash
bash tests/test_collections.sh
```

Most collection files point to real datasets outside this repository.  The
runner treats a missing input file as `SKIP`; that is expected on a fresh clone.
A non-file-related `FAILED` result is worth investigating.  Keep the generated
output outside version control (for example `./example-output` or `/tmp`)—the
repository intentionally ignores raw MRI and NIfTI output files.

## Troubleshooting checklist

| Symptom | First thing to check |
| --- | --- |
| `No module named ...` | Activate the environment and rerun `python -m pip install -e . pytest`. |
| File-not-found error | Use an absolute path in JSON and confirm that your signal, noise, and reference files exist. |
| Coil-count or array-shape error | Ensure signal, noise, and reference share the same last-axis coil count and follow the documented axis order. |
| SENSE/GRAPPA error | Confirm acceleration, `acl`, and reference/ACS data are supplied; try RSS first to isolate file-reading issues. |
| Output has incorrect physical placement | Provide trustworthy spacing, origin, and direction metadata rather than relying on defaults. |
| FA-corrected output is mostly zero | Confirm the FA map is in **degrees**, overlaps the SNR field of view, and does not contain near-zero values in the anatomy. |

For deeper implementation details, supported Siemens versions, and the
MATLAB-port comparison, read `README.md`.  For contribution and broader
collection-testing guidance, read `CONTRIBUTING.md`.
