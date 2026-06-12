# Example paper-derived queries (SME reference)

## Examples (query → expected node)

- "calibrated spectral data for Saturn's rings from Cassini UVIS" → RMS
- "Mars 2020 PIXL data" → GEO
- "Cassini magnetosphere" → PPI
- "Mars HiRISE images of Jezero crater" → GEO (not IMG)
- "Cassini ISS images of Mimas 2009-2010" → RMS (Stage 2 OPUS granule filter)
- "OSIRIS-REx OVIRS spectra of Bennu" → SBN
- "MRO MCS temperature profiles" / "MAVEN atmosphere" → ATM
- "LRO NAC images of the Apollo 11 site" → LROC
- "SPICE kernels for Cassini" → NAIF
- "MESSENGER MAG calibrated data at Mercury" → PPI
  - Example PDS3 id cited: `MESS-E_V_H_SW-MAG-3-CDR-CALIBRATED-V1.0`

## How these are used (as stated)

- Router few-shot examples.
- Evaluation seeds for offline benchmarking.

## Notes / unknowns

- Ground-truth dataset identifiers per example: TBD (beyond the single explicit MESSENGER MAG example).
- Where the eval set lives and how it is maintained: see `contexts/open-questions-2_1.md`.
