LAMOST Data Access and Spectrum Analysis
==========================================

These examples use ``astroquery.nadc.lamost``, not ``pylamost``.
Install the dependencies described in the `README <README.md>`_ first.
Interface parameters and service limitations are documented in
`the module documentation <https://github.com/china-vo/astroquery/blob/987490c7ced2dc8713bf9da4e65aeea9972fb9af/docs/nadc/lamost.rst>`_.

Inspecting Request Parameters
-----------------------------

This section uses the local ``nadc-lamost-new-module`` checkout with the
stable POST payload preview change. That change is not included in the
commit currently pinned in ``requirements.txt``. Install the neighbouring
checkout using ``python -m pip install -e ../astroquery`` as described in the
README before running these examples.

The examples below are independent of the Setup section and do not send data
queries. ``get_query_payload=True`` returns request parameters, not catalog
results. Native POST previews separate the JSON body from the URL parameters:

.. code-block:: python

   from astroquery.nadc.lamost import LamostClass

   anonymous = LamostClass(token='', data_release='dr10', sub_version='v2.0')
   payload = anonymous.query_catalog(
       'combined', columns=['obsid', 'ra', 'dec'], max_rows=5,
       get_query_payload=True)
   print(payload['json']['rows'])  # 5
   print(payload['json']['showcol'])  # ['obsid', 'ra', 'dec']
   print(payload['params'])  # {}

The same structure is returned when a token is configured. This placeholder
is used only to demonstrate redaction; it is not a valid access token:

.. code-block:: python

   authenticated = LamostClass(
       token='example-token', data_release='dr10', sub_version='v2.0')
   payload = authenticated.query_catalog(
       'combined', columns=['obsid', 'ra', 'dec'], max_rows=5,
       get_query_payload=True)
   print(payload['json']['rows'])  # 5
   print(payload['params'])  # {'token': '<redacted>'}

A redacted token indicates that a token would be sent; it does not establish
that the service will accept it or identify how it was configured.

GET previews, including SQL and related-observation lookups, are flat:

.. code-block:: python

   sql_payload = anonymous.query_sql(
       'SELECT obsid, ra, dec FROM combined LIMIT 5', get_query_payload=True)
   print(sql_payload['sql'])  # SELECT obsid, ra, dec FROM combined LIMIT 5
   repeat_payload = anonymous.query_repeat_observations(
       obsid=101001, get_query_payload=True)
   print(repeat_payload['obsid'])  # 101001 (a string)

Catalog queries that use SQL, including cone searches and legacy-release
queries, also return the flat SQL format. Their row limit is part of the SQL
statement, so there is no ``rows`` key. Structured SQL previews may fetch
field metadata, and object names may require online coordinate resolution.
The three code blocks above use neither and can run offline.

Setup
-------

For catalog, LRS and MRS examples, run setup below first, then the code blocks
in the desired section in order in one Python session. These examples use
DR10/v2.0 and require internet access. The activity example is independent
and uses DR7/v2.0.

Start Python from the repository directory. This setup writes downloaded and
generated files into ``output/``, which is ignored by Git. Examples with
``overwrite=True`` replace existing files. Configure authentication as
explained in the `README <README.md#authentication>`_ if required.

.. code-block:: python

   import os
   from pathlib import Path
   from astropy.coordinates import SkyCoord
   from astroquery.nadc.lamost import LamostClass

   output = Path('output')
   output.mkdir(exist_ok=True)
   os.chdir(output)
   lamost = LamostClass(data_release='dr10', sub_version='v2.0')
   coord = SkyCoord(10.0004738, 40.9952444, unit='deg', frame='icrs')

Catalog Pagination and Export
-------------------------------

Query methods return one page. To retrieve a larger sample, keep the selection,
page size, and sort order fixed while advancing ``page``. For example:

.. code-block:: python

   page = 1
   while True:
       matches = lamost.query_spectra(
           coord, '0.2 deg', columns=['obsid', 'ra', 'dec'],
           sort_by='obsid', max_rows=100, page=page)
       if len(matches) == 0:
           break
       matches.write(f'lamost-page-{page}.ecsv', format='ascii.ecsv', overwrite=True)
       page += 1

These are separate requests, not an atomic snapshot of a changing catalog.
Errors propagate instead of being treated as an empty final page. For a
single returned table, use ``Table.write`` with the desired local format;
the service's ``output_format`` controls transport rather than file saving.

LRS Export and Smoothing
--------------------------

Download one LRS observation for local processing:

.. code-block:: python

   lrs_files = lamost.get_spectra(176604010, resolution='low', verify='silentfix')
   try:
       lrs_files[0].writeto('lrs-spectrum.fits', overwrite=True, output_verify='silentfix')
   finally:
       for spectrum in lrs_files:
           spectrum.close()

Astropy may fix structural header issues when writing; the saved FITS is not
a byte-for-byte copy of the response. Wavelengths are in angstroms; flux
values retain the archive units and normalization. Consult the FITS headers
before interpreting them as calibrated flux. The simplified arrays exclude
inverse variance and quality masks; read those from the original FITS for
pixel selection and error propagation.

For example, export LRS arrays locally:

.. code-block:: python

   from astropy.table import Table
   from astroquery.nadc.lamost import parse_lrs_spectrum

   wavelength, flux = parse_lrs_spectrum('lrs-spectrum.fits')
   Table({'wavelength': wavelength, 'flux': flux}).write(
       'spectrum.csv', format='ascii.csv', overwrite=True)

Smoothing is a separate analysis choice. This seven-pixel median uses
zero-padded edges; setting ``window = 15`` selects a wider filter. Neither
result replaces the original flux:

.. code-block:: python

   import numpy as np

   window = 7
   samples = np.lib.stride_tricks.sliding_window_view(
       np.pad(flux, window // 2), window)
   smoothed_flux = np.median(samples, axis=-1)

MRS Batch Reading and Plotting
--------------------------------

One MRS FITS can contain multiple exposures; preserve extension labels and
do not treat ``obsid`` as a unique identifier for every exposure row.

First download one MRS observation from the same DR10/v2.0 service:

.. code-block:: python

   mrs_files = lamost.get_spectra(1007903112, resolution='medium')
   try:
       mrs_files[0].writeto('mrs-spectrum.fits', overwrite=True, output_verify='fix')
   finally:
       for spectrum in mrs_files:
           spectrum.close()

For several local MRS files, retain each outcome so missing or invalid data
does not disappear from the sample. This example intentionally includes a
nonexistent filename to demonstrate the failure record:

.. code-block:: python

   from astropy.table import Table
   from astroquery.nadc.lamost import parse_mrs_spectrum

   spectra, rows = [], []
   for filename in ['mrs-spectrum.fits', 'missing-mrs.fits']:
       try:
           spectrum = parse_mrs_spectrum(filename)
       except (OSError, EOFError, ValueError) as error:
           spectra.append(None)
           rows.append((filename, 'ERROR', f'{type(error).__name__}: {error}'))
       else:
           spectra.append(spectrum)
           rows.append((filename, 'COMPLETE', ''))
   manifest = Table(rows=rows or None, names=('Local Path', 'Status', 'Message'),
                    dtype=(str, str, str))
   print(manifest['Local Path', 'Status', 'Message'])

This loop preserves input order, records expected file and validation errors,
and propagates unexpected exceptions. It does not repair or modify the files.

Plot a local MRS file with Matplotlib, retaining the extension labels:

.. code-block:: python

   import matplotlib.pyplot as plt
   from astroquery.nadc.lamost import parse_mrs_spectrum

   fig, ax = plt.subplots()
   for name, spectrum in parse_mrs_spectrum('mrs-spectrum.fits').items():
       ax.plot(spectrum['wavelength'], spectrum['flux'], label=name)
   ax.set(xlabel='Wavelength [Angstrom]', ylabel='Flux')
   ax.legend()
   fig.savefig('spectrum.png')
   plt.close(fig)

One-observation Activity Example
----------------------------------

The `standalone script <lamost_activity.py>`_ selects DR7/v2.0,
retrieves metadata and a spectrum for observation 54901214, verifies the
metadata and FITS ``OBSID``, and computes the dimensionless Ca II H&K index
``S_L`` from `Zhang et al. (2022) <https://arxiv.org/abs/2209.15255>`_. It uses
only NumPy, Astropy, and astroquery. Configure authentication as described in
the `README <README.md#authentication>`_ before running this example if anonymous access is rejected. DR7/v2.0
may require a valid token even when the DR10/v2.0 examples work anonymously.
The observed DR7 metadata and FITS endpoints are public, but ``get_metadata``
also uses SQL to obtain column types; that SQL request requires authentication.
For a standalone script, set ``ASTROQUERY_NADC_LAMOST_TOKEN`` in its
environment; a ``conf.token`` assignment in another Python process does not
carry over to the script.
``get_metadata`` normalizes the legacy DR7 response into a one-row table
with named fields. The example uses its observation ID and radial velocity.

Run the script from the repository directory to retrieve metadata and the spectrum:

.. code-block:: console

   $ python lamost_activity.py
   OBSID=54901214: S_L=0.180373854
   Published S_L=0.18038 +/- 0.003597

For an existing FITS file of the same observation (54901214), run without
networking:

.. code-block:: console

   $ python lamost_activity.py --filename 54901214.fits --rv -24.78

The archive RV is -24.78 km/s. The example divides the vacuum wavelengths by
``1 + RV/c`` before applying the paper's bands: 20-Angstrom rectangular
continua centered at 4002.20 and 3902.17 Angstroms, and triangular cores
centered at 3969.59 and 3934.78 Angstroms with FWHM 1.09 Angstroms. It uses
linear interpolation and trapezoidal weights, then evaluates
``S_L = (1.8 * 8 * 1.09 / 20) * (H + K) / (R + V)``.

The script checks its result against the independently archived value
``0.1803738541038765`` to ``1e-7``; this computational tolerance is separate
from the published uncertainty. It is a fixed-observation example, not an
uncertainty propagation or Mount Wilson calibration. All pixels supporting a
bandpass, including the interpolation neighbours just outside its edges, must
have finite positive flux; invalid pixels, RV or results raise ``ValueError``
naming the band and pixel. Assess archive pixel masks and selection criteria
for other targets.

For batch measurements, start a new Python session from the repository
directory and import ``measure_files`` from ``lamost_activity.py``. Supply the archive RV in km/s and expected integer OBSID
for each file; each identity is checked before computing its index:

.. code-block:: python

   from lamost_activity import measure_files

   manifest = measure_files([
       ('missing.fits', -24.78, 54901214),
       ('tests/data/activity_dr7_excerpt.fits.gz', -24.78, 54901214),
   ])
   print(manifest['Local Path', 'OBSID', 'Status', 'Message', 'S_L'])
   successful = manifest[manifest['Status'] == 'COMPLETE']

Like the MRS loop above, this returns one status row per input, continues
after file or validation errors, and propagates unexpected exceptions. Failed
``S_L`` values are masked and displayed as ``--``. No replacement index is
computed for a failed sample. Batch values are not compared to the fixed
observation's reference index; the script's original single-observation command
continues to perform that comparison.
