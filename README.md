# Astroquery NADC LAMOST Examples

Examples of querying LAMOST data and analysing spectra with
`astroquery.nadc.lamost`. This repository does not use or depend on `pylamost`.

The examples cover catalog export, LRS spectrum export and smoothing, MRS
batch reading and plotting, and a Ca II H&K activity-index calculation.
See [the tutorial](TUTORIAL.rst) for the complete workflows.

## Install

Use Python 3.10 or newer. From the repository directory:

```bash
python -m venv .venv
source .venv/bin/activate
python -m pip install -r requirements.txt
```

On Windows, activate with `.venv\Scripts\Activate.ps1` instead.
Git is required for installation from the source repository.

`astroquery.nadc.lamost` is under review in
[astroquery PR #3666](https://github.com/astropy/astroquery/pull/3666).
`requirements.txt` pins the tested PR commit
`987490c7ced2dc8713bf9da4e65aeea9972fb9af`; an ordinary PyPI installation
must not be assumed to contain this module yet. NumPy and Astropy are
astroquery dependencies; Matplotlib is used for plotting.

For local development against the neighbouring astroquery checkout, use this
instead of `requirements.txt`:

```bash
python -m pip install -e ../astroquery matplotlib
```

Check that the checkout is on `nadc-lamost-new-module` before using that option.

## Reproduce the activity index offline

Run from this repository directory:

```bash
python lamost_activity.py --filename tests/data/activity_dr7_excerpt.fits.gz --rv -24.78
```

Expected output:

```text
OBSID=54901214: S_L=0.180373854
Published S_L=0.18038 +/- 0.003597
```

The included FITS is an excerpt covering the required bandpasses, not a full
spectrum. Its provenance, source hash and numerical reference are recorded
in [tests/data/reference.json](tests/data/reference.json).
The numerical check is not an uncertainty estimate or a Mount Wilson calibration.

## Authentication

The online example uses DR7/v2.0; its metadata schema query may require a
LAMOST/NADC token even when DR10/v2.0 examples work anonymously.
Set `ASTROQUERY_NADC_LAMOST_TOKEN` in the shell that runs the examples.
Do not put real tokens in source files, notebooks, or committed output.
A `conf.token` assignment in another Python process does not carry over.

To retrieve the observation and compute its index online:

```bash
python lamost_activity.py
```

Service availability and access permissions depend on the selected release.
The offline example does not require a token or internet access.

## Tests

The migrated tests use local FITS fixtures and synthetic spectra. They do not
import `astroquery`'s internal test helpers or require online archive access.

```bash
python -m pip install pytest
python -m pytest tests -q
```

## Origin and licence

The tutorial, activity script and tests were extracted from astroquery PR #3666
at commit `987490c7ced2dc8713bf9da4e65aeea9972fb9af`.
The activity example follows [Zhang et al. (2022)](https://arxiv.org/abs/2209.15255).
See [LICENSE.rst](LICENSE.rst) for the original BSD 3-Clause licence and notices.
