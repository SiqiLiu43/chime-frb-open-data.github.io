*Author: Marcus Merryfield*

The injections dataset used for catalog 1 is separated into two files, available in the [data releases section](https://chime-frb-open-data.github.io/data-releases/). The first encompasses the full set of 5 million synthetic FRBs which were generated from population distributions based on The First CHIME/FRB Catalog. The second encompasses a subset of these 5 million bursts (~85,000 total) which were actually injected into the live CHIME/FRB intensity data stream. The first file is available as an `hdf5` file, while the second is available as a pickle file intended to be read by the [`pandas`
library](https://pandas.pydata.org/) as a `DataFrame`.

???+ Example "Read in the 5 million synthetic pulses"
    ```python
    import h5py

    fn = "chimefrb_catalog1_injections_full.h5"
    datafile = h5py.File(fn, 'r')

    # datafile.keys() will show the sets of data available:
    # ['frb', 'freq', 'injection_format', 'to_inject', 'to_inject_fit_spec_coeffs']

    # Create datasets: 'frb'
    dset_frb = datafile['frb']
    data_frb = dset_frb[()]

    # 'freq'
    dset_freq = datafile['freq']
    data_freq = dset_freq[()]

    # 'injection_format'
    dset_inj_format = datafile['injection_format']
    data_inj_format = dset_inj_format['frb'][()]

    # 'to_inject'
    dset_to_inj = datafile['to_inject']
    data_to_inj = dset_to_inj[()]

    # 'to_inject_fit_spec_coeffs'
    dset_speccoeffs = datafile['to_inject_fit_spec_coeffs']
    data_speccoeffs = dset_speccoeffs[()]
    ```

??? Hint "MetaData descriptions"
    Description for each of the datafile keys:

    - `frb`: The dataset for all 5 million FRBs (hint: empty tuples retrieve all data for `hdf5` datasets). The `data_frb.dtype` shows which attributes are available in this dataset:
        - `loc_ind`: The index of the sky location chosen for a given FRB in fixed telescope coordinates. These indexes are drawn to correspond with locations where we expect the beam response to be significant enough for possible detection of synthetic bursts.
        - `dm`: The DM of a given FRB, in pc/cm^3.
        - `width`: The intrinsic width of a given FRB, in s.
        - `scat_ref`: The scattering time of a given FRB, in s, at a reference frequency of 600 MHz.
        - `spec_coeffs`: The array of three spectral coefficients for an FRB. The three indices of `spec_coeffs` give the log of the bandaveraged fluence of the spectrum normalized by the band mean fluence of the spectrum (index 0), spectral index (index 1), spectral running (index 2). Note the latter two values are referenced at 400 MHz.
        - `x`: The CHIME/FRB beam model x coordinate for the given FRB
        - `y`: The CHIME/FRB beam model y coordinate for the given FRB
        - `ra`, `dec`, `to_inf`, and `toa_inf` are unused
    - `freq`: The dataset of 1024 frequencies (~400-800 MHz) used for determining the spectral properties for each FRB.
    - `injection_format`: A group which has one key containing a dataset (`datafile['injection_format']['frb']`) with information for the 5 million FRBs in the format expected by the injection API. The `data_inj_format.dtype` shows which attributes are available in this dataset:
        - `beam_no`: The CHIME/FRB beam number for the injection. Given as -1 for the majority of events, as they were not put up for injection. For the frbs which were put up for injection, there are four beam columns: the zeroeth column has beams `0-255`, the first has beams `1000-1255`, the second has beams `2000-2255`, and the third has beams `3000-3255`.
        - `injection_program_id`: The name of the injection program used to identify sets of injected bursts. In this data, the id has not been filled yet, and it has been temporarily populated with the index.
        - `beam_x` and `beam_y`: Same as `x` and `y` for the `frb` key.
        - `dm`: Same as in the `frb` key.
        - `tau_1_ghz`: The scattering time referenced to 1 GHz, in ms.
        - `pulse_width_ms`: The intrinsic width of a given FRB, in ms.
        - `fluence`: The band mean fluence of a given FRB, in Jy ms.
        - `spindex` and `running`: Same as index 1 and 2 of `spec_coeffs` for the `frb` key.
        - `to_inject`: The boolean stating whether or not the burst made the cut into the `to_inject` dataset.
        - `injected`: The boolean stating whether a given burst has yet been injected. Note that since this information was updated in a different file, all values here are `False`.
    - `to_inject`: The dataset of ~97,000 bursts which are a subset of the 5 million that passed the SNR estimate cut to go up for injection. Note only ~85,000 of these were injected, as some injections were lost due to networking issues.
        - `frb_ind`: The index in the `injection_format` corresponding to a given `to_inject` burst.
        - `beam_id`: Same as `beam_no` in the `injection_format` key.
        - `snr_estimate`: The estimated signal-to-noise of the injected events, determining whether the event would be put up for injection, calculated using the radiometer equation. For the `to_inject` dataset, the cutoff SNR was set to 20. While this is significantly higher than the SNR cutoff of 9 used in L1 for the majority of the First CHIME/FRB Catalog, the adjustment was necessary because the estimated SNR using the idealized assumptions in the radiometer equation was far too optimistic compared to actual detection SNRs in initial tests.
        - `fluence_spectrum`: An array with 1024 fluence values from 400 to 800 MHz, giving the time-integrated fluence spectrum of a synthesized FRB. This spectrum is modulated by the beam.
    - `to_inject_fit_spec_coeffs`: The array of fit spectral coefficients for the ~97,000 `to_inject` events. The array indices are the same as given for `spec_coeffs` in the `frb` dataset. These fits were used as the best estimate of what spectral coefficients would be recovered from CHIME/FRB intensity data, based on the `fluence_spectrum` of the `to_inject` bursts.


???+ Example "Read in the ~85,000 injected events"
        
    ```python
    import numpy as np
    import pandas as pd

    # Read in the pickle file as a pandas DataFrame
    fn = "chimefrb_catalog1_injections.p"
    data = pd.read_pickle(fn)

    # Now separate the data into categories: detected, non-detected, and RFI
    # Note that in the CHIME/FRB Catalog 1 analysis, an rfi_threshold of 7
    # and a high_snr_override of 30 were used.
    snr_threshold = 9.
    rfi_threshold = 5.
    high_snr_override = 100.

    # Make a detection mask following CHIME/FRB pipeline logic
    detected_mask = np.logical_and.reduce(
        (
            data.bonsai_snr.values > snr_threshold,
            np.logical_or(
                data.l2_rfi_grade.values > rfi_threshold,
                data.bonsai_snr.values > high_snr_override
            )
        )
    )

    # Create arrays of detected & non-detected injections
    det = data[detected_mask]
    nondet = data[~detected_mask]

    # Make an RFI mask, where RFI are the subset of non-detections which had
    # SNRs above the SNR threshold
    rfi_mask = np.logical_and(
        pd.notna(nondet.l2_rfi_grade),
        nondet.bonsai_snr.values > snr_threshold
    )

    # Create array of RFI injections & update non-detected injections
    rfi = nondet[rfi_mask]
    nondet = nondet[~rfi_mask]
    ```

??? Hint "MetaData descriptions"
    Descriptions for each of the columns in the `DataFrame` (listed via `data.keys()`):

    - `beam_x`: The x position of the injection in CHIME/FRB beam coordinates. The CHIME/FRB beam model is called at an (x,y) coordinate pair to include the forward modelled effect of the beam on the synthetic pulses.
    - `beam_y`: The y position of the injection in CHIME/FRB beam coordinates.
    - `beams`: An array of CHIME/FRB beam numbers which the synthetic pulse was injected into. For the First CHIME/FRB Catalog, injections were performed in single beams only. There are four beam columns: the zeroeth column has beams `0-255`, the first has beams `1000-1255`, the second has beams `2000-2255`, and the third has beams `3000-3255`.
    - `bonsai_snr`: The signal-to-noise ratio (SNR) reported by CHIME/FRB's L1 pipeline. For the majority of the observing period in the First CHIME/FRB Catalog, the detection threshold was at an SNR of 9.
    - `dm_det` and `dm_inj`: The detection DM as reported by the L1 pipeline (`_det`, if available) and the synthetic pulse's injected DM (`_inj`), in pc/cm^3.
    - `dm_err_det`: The error in the DM as reported by the L1 pipeline, in pc/cm^3. Note that DM errors are discrete as a function of L1 tree index.
    - `dm_gal_ne_2001_max` and `dm_gal_ymw_2016_max`: The maximum DM along the line of site estimated by the NE2001 and YMW16 Galactic DM models, in pc/cm^3. The detected position for synthetic pulses is approximately the center of the beam which they were injected into, since the injections were only performed in single beams.
    - `fluence_jy_ms_inj`: The estimated injection fluence of the synthetic pulse, in Jy ms.
    - `l1_rfi_grade`: The RFI grade (scale of 10) reported by the L1 pipeline.
    - `l2_rfi_grade`: The RFI grade (scale of 10) reported by the L2/L3 pipeline. The threshold for a detection to be considered astrophysical (as opposed to RFI) is 5.
    - `pos_dec_deg_det` and `pos_ra_deg_det`: The approximate RA and Dec positions of the detections, in degrees. As injections were performed in single beams, represents approximate the RA and Dec of the beam center at the time of detection.
    - `pulse_width_ms_det` and `pulse_width_ms_inj`: The detected pulse width as reported by the L1 pipeline and the synthetic pulse's injected intrinsic width, in ms. Note that L1 does not record which width bin had the highest SNR detection, so the detected width is reported as 2x the size of the time bins in the detection tree.
    - `spectral_index_det` and `spectral_index_inj`: The detected spectral index as reported by the L1 pipeline and the synthetic pulse's injected spectral index. L1 only reports two possible spectral indices: +3, or -3.
    - `spectral_running_inj`: The synthetic pulse's injected spectral 'running', using the running power-law with which CHIME/FRB models real detections. Note that since intensity data is not saved for injected events and L1 only considers a regular power-law weighting there is no detected value for spectral running.
    - `t_err_ms_det`: The approximate timing error from L1 for the detected pulse, in ms.
    - `t_utc_det`, `t_utc_expected`, and `t_utc_inj`: The UTC time of the detection as reported by the L1 pipeline, the expected UTC time of synthetic pulse detection before injection, and the UTC time which the synthetic pulse was actually injected.
    - `tau_1_ghz_ms_inj`: The scattering index at 1 GHz of the synthetic pulse, in ms.
    - `tree_index_det`: The tree index of the detection. Tree index is indexed starting at zero, going to a maximum of four, with each sequential tree increasing the temporal binning by a power of two.