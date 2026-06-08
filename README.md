# py-nist-sp800-22

A correct, pure-Python implementation of the **NIST SP 800-22 Rev. 1a** statistical randomness test suite.

## Features

- **15 working tests** — all verified against true random data (`os.urandom`)
- **Pure Python** — only `numpy` needed for the DFT/Spectral test (optional)
- **No sampling** — tests run on the entire file, not just 1M-bit fragments
- **Self-validation** — built-in `--validate` mode tests against `os.urandom()`
- **Simple API** — single file, usable as CLI tool or Python library

## Validation

All tests have been validated against 5 independent data sources:

| Input | Expected | Result |
|-------|----------|--------|
| `os.urandom()` (CSPRNG) | All PASS | 17/17 PASS |
| AES-256 (7-Zip) | All PASS | 17/17 PASS |
| Turbine V5 cipher | All PASS | 17/17 PASS |
| SCHFM2 cipher | All PASS | 17/17 PASS |
| Raw JPEG (negative control) | All FAIL | 0/17 PASS |

## Installation

No installation needed. Just copy `nist_sp800_22.py` to your project.

Optional dependency for the DFT/Spectral test:
```
pip install numpy
```

## Usage

### Command line

```bash
# Test a file
python nist_sp800_22.py encrypted.bin

# Skip header bytes (e.g., 70 bytes for Turbine .tur files)
python nist_sp800_22.py data.tur 70

# Self-validation (test against os.urandom)
python nist_sp800_22.py --validate
```

### Python API

```python
from nist_sp800_22 import run_all_tests, monobit_frequency, bytes_to_bits
import os

# Run all tests on random data
data = os.urandom(1_000_000)
results = run_all_tests(data, label="My test data")

# Run individual tests
bits = bytes_to_bits(data)
p = monobit_frequency(bits)
print(f"Monobit p-value: {p:.6f}")
```

## Tests included

### NIST SP 800-22 Tests (1-13)

| # | Test | What it checks |
|---|------|---------------|
| 1 | Monobit Frequency | Equal proportion of 0s and 1s |
| 2 | Block Frequency (M=128) | Uniform distribution within blocks |
| 3 | Runs Test | Expected number of bit transitions |
| 4 | Longest Run of Ones | Longest consecutive 1s in blocks |
| 5 | Cumulative Sums (forward) | Random walk stays near zero |
| 6 | Cumulative Sums (backward) | Same, in reverse |
| 7 | Approximate Entropy (m=10) | Overlapping pattern frequencies |
| 8 | Serial Test (m=16) | All m-bit patterns equally likely |
| 9 | Maurer's Universal | Compressibility of the sequence |
| 10 | DFT / Spectral | Periodic features (requires numpy) |
| 11 | Binary Matrix Rank | Rank distribution of 32x32 matrices |
| 12 | Non-Overlapping Template | Specific pattern occurrence frequency |
| 13 | Linear Complexity | Shortest LFSR via Berlekamp-Massey |

### Supplementary Tests (14-17)

| # | Test | What it checks |
|---|------|---------------|
| 14 | Chi-Squared Byte | Uniform byte value distribution |
| 15 | Compression Ratio | Incompressibility (zlib) |
| 16 | Serial Correlation | Independence of consecutive bytes |
| 17 | Shannon Entropy | Information density (bits/byte) |

## Note on the Linear Complexity test

The pi values published in NIST SP 800-22 (Table 2.10) are **incorrect** — they do not match the theoretical distribution of linear complexity for any block size M. This implementation computes the correct pi values from the exact probability distribution using dynamic programming over the Berlekamp-Massey state transitions. Additionally, the T statistic formula uses `(-1)^M` (not `(-1)^(M+1)` as in some implementations). These corrections were verified empirically: the test now passes for `os.urandom()` data and fails for non-random data (raw JPEG).

With the incorrect pi values, the test returns p=0.000 (FAIL) for **every input** — including perfectly random data from `os.urandom()`. Empirical testing confirms: 0 out of 50 random trials pass, and no combination of non-random inputs produces a PASS either. The test becomes a dead test with zero diagnostic value, which is why it is silently skipped in most implementations.

This affects virtually every existing NIST test suite — including `nistrng`, the original NIST STS C code, and most ports thereof.

## Interpreting results

- **p-value >= 0.01**: PASS — no evidence of non-randomness
- **p-value < 0.01**: FAIL — statistically significant deviation from randomness
- A single marginal failure (p near 0.01) on an otherwise good dataset may be a statistical fluke. Multiple failures indicate a real problem.

## License

MIT License — see [LICENSE](LICENSE).

## Author

Reinhard Jesolowitz ([ReinhardJesolowitz24](https://github.com/ReinhardJesolowitz24))
