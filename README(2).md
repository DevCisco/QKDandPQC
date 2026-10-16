# QKD v2.1 — BB84 · SSP · E91 with a post-quantum cryptographic stack

A [NetSquid](https://netsquid.org/) simulation of three Quantum Key Distribution
protocols, with the complete classical pipeline: Cascade reconciliation, key
verification, decoy states, privacy amplification and post-quantum
authentication of the classical channel via
[liboqs](https://github.com/open-quantum-safe/liboqs).

The project does not stop at showing that BB84 "works": it measures the cost of
every stage of the pipeline and checks that the security guarantees it claims
are actually enforced by the code.

---

## Contents

- [What it includes](#what-it-includes)
- [Requirements](#requirements)
- [Installation](#installation)
- [Running](#running)
- [Architecture](#architecture)
- [Main parameters](#main-parameters)
- [Design choices](#design-choices)
- [Reference results](#reference-results)
- [Known limitations](#known-limitations)
- [Troubleshooting](#troubleshooting)
- [What changed in v2.1](#what-changed-in-v21)
- [References](#references)

---

## What it includes

**Three QKD protocols**

| Protocol | Paradigm | Bases | Sifting | QBER threshold | Security test |
|---|---|---|---|---|---|
| BB84 | prepare-and-measure | 2 (Z, X) | ~50% | 11.0% | QBER |
| SSP (six-state) | prepare-and-measure | 3 (Z, X, Y) | ~33% | 12.6% | QBER |
| E91 | entanglement-based | 3 angles | ~22% | 11.0% | CHSH (S > 2, reported as S ± σ) |

**Classical stack**

- **Complete Cascade** (Brassard & Salvail 1994) — adaptive block
  `k₁ = ⌊0.73/QBER⌋`, full back-propagation of corrections, parities never
  disclosed twice. Adapted to short keys (`k₁ ≤ n/4`, blocks `≤ n/2`, 6 passes).
  Compared with a 2-pass version on identical data.
- **Key verification** — 32-bit hash tag; sessions where Alice and Bob end with
  different keys are aborted.
- **Decoy states** (Hwang 2003; Lo-Ma-Chen 2005) — LMC key-rate bound with an
  attenuated laser source, countermeasure to photon-number splitting.
- **Privacy amplification** — Toeplitz hashing with a fresh random seed per
  session, capped by the Leftover Hash Lemma.
- **AES-256-GCM** — final encryption; Bob decrypts with his independently
  derived key.

**Post-quantum stack (liboqs)**

- **ML-DSA-65** (FIPS 204) — signs all four classical messages: basis
  announcement, Cascade parities, privacy-amplification seed, key confirmation.
- **ML-KEM-768** (FIPS 203) — K1 component of the hybrid key.
- Session key: `K = HKDF-SHA256(K1 ‖ K2, session_id)`, with K2 the QKD key
  **after** privacy amplification.

**Physical models**

Fibre attenuation (Beer-Lambert), PMD, dark counts, detector timing jitter
(SNSPD vs InGaAs), with a per-source QBER breakdown.

**Simulated attacks**

- Intercept-resend with variable interception rate (η).
- Active man-in-the-middle on the quantum **and** classical channel.

---

## Requirements

- Python 3.11 (NetSquid wheels are not available for every Python version;
  3.13/3.14 are not recommended)
- **NetSquid** — requires a free account on [netsquid.org](https://netsquid.org/)
  (not on public PyPI)
- `numpy`, `scipy`, `matplotlib`, `cryptography`, `ipykernel`
- `liboqs-python` (optional but recommended — see below)
- `pyspx` (signature fallback when liboqs is not available)

The code runs with both NumPy 1.x (required by NetSquid) and NumPy 2.x.

---

## Installation

```bash
python3.11 -m venv venv_netsquid
source venv_netsquid/bin/activate

pip install numpy scipy matplotlib cryptography ipykernel pyspx
pip install netsquid --extra-index-url https://pypi.netsquid.org   # asks for your NetSquid credentials
```

If your system Python is not 3.11, [uv](https://docs.astral.sh/uv/) can create
the environment with the right version: `uv venv --python 3.11 venv_netsquid`.

Once the environment works, freeze it so it can be rebuilt identically:

```bash
pip freeze > requirements.txt
```

### liboqs

> **Beware of the PyPI name clash.** The package `oqs` is *Open Quick Script*,
> an expression interpreter with **no relation** to post-quantum cryptography.
> Both install a module importable as `oqs` and overwrite each other. The
> correct package is `liboqs-python`. The notebook checks the API, not just the
> import.

```bash
pip uninstall -y oqs            # removes the namesake, if present
pip install liboqs-python
python -c "import oqs; print(oqs.oqs_version())"
```

On first import liboqs-python compiles the C library, which takes a few minutes
and needs `cmake` and a compiler. If it fails with `No oqs shared libraries
found`:

```bash
pip install cmake ninja          # or: sudo apt install cmake ninja-build build-essential
rm -rf ~/_oqs                    # removes the failed attempt
python -c "import oqs; print(oqs.oqs_version())"
```

If compilation stops at `Could NOT find OpenSSL`, install the headers with
`sudo apt install libssl-dev`, or build liboqs without OpenSSL:

```bash
git clone --depth=1 https://github.com/open-quantum-safe/liboqs.git liboqs-src
cmake -S liboqs-src -B liboqs-build -GNinja \
      -DBUILD_SHARED_LIBS=ON -DOQS_BUILD_ONLY_LIB=ON -DOQS_USE_OPENSSL=OFF \
      -DCMAKE_INSTALL_PREFIX=$HOME/.local
ninja -C liboqs-build && ninja -C liboqs-build install
export LD_LIBRARY_PATH=$HOME/.local/lib:$LD_LIBRARY_PATH
```

With this route `LD_LIBRARY_PATH` must be exported **before** starting Jupyter
or VS Code: an `export` inside a cell has no effect on a running kernel.

Quick check that signatures work end to end:

```bash
python -c "import oqs; s=oqs.Signature('ML-DSA-65'); pk=s.generate_keypair(); print(oqs.Signature('ML-DSA-65').verify(b'x', s.sign(b'x'), pk))"
```

### Without liboqs

The notebook still runs: signatures fall back to pyspx (SPHINCS+) and the KEM to
X25519. Results are then **not post-quantum on K1**, and the code says so
explicitly — at runtime (`quantum_safe_k1=False`) and in the conclusions, which
switch to a version with a warning. `print_oqs_diagnostic()` prints the exact
cause of the fallback. With neither liboqs nor pyspx, sessions that request
authentication are marked as not secure.

---

## Running

Open `progettobb84_v2.1.ipynb`, select the `venv_netsquid` kernel and run all
cells. The notebook is split into:

- **definition cells** (Modules 1–10), each preceded by a text cell with the
  theory and formulas — they define functions and run no simulation;
- **one cell per experimental phase** (Phase 0 → Phase 9), which run the
  experiment and keep their results in global variables, so a single phase can
  be re-run without starting over.

A full run takes a few minutes and produces:

- the text output of every phase;
- `qkd_output/qkd_physical_model.png` — 4 panels on the physical model;
- `qkd_output/qkd_dashboard_final.png` — 12-panel dashboard.

`qkd_output/` is created next to the notebook (works on Windows, macOS, Linux
and WSL).

### Programmatic use

```python
safe, key, stats = run_qkd_protocol(
    num_bits=3000,
    distance=25.0,
    noise_rate=0.03,
    protocol="BB84",          # "BB84" | "SSP" | "E91"
    eve_present=False,
    use_full_cascade=True,
    cascade_passes=CASCADE_PASSES,   # 6 by default
    use_pq_auth=True,         # PQ signature of all classical messages
    use_decoy_states=True,
    use_mlkem_hybrid=True,    # K = HKDF(K1_MLKEM ‖ K2_QKD)
    jitter_detector="SNSPD",  # "SNSPD" | "InGaAs"
    pa_strict=False,          # True = reject sessions with <128 extractable bits
)
```

`stats` contains the raw and post-reconciliation QBER, disclosed bits,
`key_verified`, privacy-amplification output (`pa_output_bits`,
`pa_available_bits`), `abort_reason` (`fotoni`, `sifting`, `QBER`, `CHSH`,
`verifica`, `PA`, `firma`) and the sub-dictionaries `decoy`, `auth`, `mlkem`.
For E91 it also contains `S_chsh`, `S_sigma` and `chsh_signif`.
`final_key_bob` holds Bob's independently derived key. The v2.0 parameter name
`use_sphincs_auth` is still accepted.

---

## Architecture

| Module | Content |
|---|---|
| 1 | Channel physics: attenuation, PMD, dark counts, jitter |
| 2A-D | BB84/SSP (shared encode/measure helpers) + Eve intercept-resend |
| 2E | E91: entangled source, angle measurements, CHSH parameter and its uncertainty |
| 3 | Complete Cascade (short-key caps) + 2-pass v1.0 baseline |
| 3B | Decoy states, LMC bound, PNS analysis |
| 3C | Post-quantum signature (liboqs) + benchmark + soundness test |
| 3D | Hybrid KEM ML-KEM-768 + HKDF derivation |
| 4 | Privacy amplification (Toeplitz, LHL ceiling, random seed) |
| 2F | Active MITM attack (quantum + classical) |
| 5 | Unified BB84/SSP/E91 runner with shared post-processing |
| 6 | AES-256-GCM encryption |
| 7 | No-cloning analysis + E91 security |
| 8A-8H | Distance trade-off, statistics, key rate, PA comparison, physical model |
| 9 | Plots: physical model and 12-panel dashboard |
| 10 | Narrative conclusions (driven by the measured data) |

---

## Main parameters

| Constant | Value | Notes |
|---|---|---|
| `FIBRE_ATTENUATION_DB_PER_KM` | 0.1419 | ultra-low-loss PSCF fibre |
| `DARK_COUNT_RATE_HZ` | 10.0 | state-of-the-art SNSPD |
| `JITTER_SNSPD_PS` | 50.0 | negligible QBER contribution at 1 GHz |
| `JITTER_INGAAS_PS` | 400.0 | ~10.6% QBER at 1 GHz — dominant factor |
| `CLOCK_PERIOD_NS` | 1.0 | 1 GHz system |
| `MU_SIGNAL` / `MU_DECOY1` | 0.5 / 0.1 | signal and decoy intensities |
| `CASCADE_PASSES` | 6 | Cascade passes |
| `CASCADE_K1_MAX_FRACTION` / `CASCADE_BLOCK_MAX_FRACTION` | 4 / 2 | initial block ≤ n/4, blocks ≤ n/2 |
| `VERIFY_TAG_BITS` | 32 | key-verification tag (counted as disclosed) |
| `SECURITY_PARAM` | 64 | bits subtracted by privacy amplification |
| `MIN_CHSH_SAMPLES` | 30 | minimum samples per CHSH correlation |
| `SIG_ALG_PRIMARY` | `ML-DSA-65` | `SLH-DSA-SHAKE-128s` as alternative |
| `KEM_ALG_PRIMARY` | `ML-KEM-768` | X25519 fallback without liboqs |

The global seed (`set_global_seed`) makes basis choices, photon losses and
Cascade permutations reproducible. Cryptographic keys are **not** seeded: they
always use the system CSPRNG, because deterministic keys would be a security
anti-pattern.

---

## Design choices

### ML-DSA-65 instead of SLH-DSA

The built-in benchmark measures, on the machine in use, a 3309 B signature
against 7856 B, and about 0.4 ms against about 1.2 s per signature. In QKD the
classical-channel overhead is not a detail: signing the four messages of a
session costs 12.9 KB with ML-DSA-65 and would cost about 31 KB with SLH-DSA,
two orders of magnitude more than the key being produced.

**Declared trade-off:** with ML-DSA-65 for signatures and ML-KEM-768 for the
KEM, both post-quantum components rest on lattices. Their security is **not
independent**. K2 (QKD) remains independent, being information-theoretic. To
diversify assumptions, set `SIG_ALG_PRIMARY = "SLH-DSA-SHAKE-128s"`.

### Cascade adapted to short keys, followed by verification

The rule `k₁ = 0.73/QBER` assumes long keys. With a few hundred bits and a low
QBER the first block becomes as large as the key and error pairs escape every
pass. Capping the blocks and using 6 passes brings the sessions with residual
errors from 8–14% to 0.3–2% (300 simulated sessions per configuration), at a
cost of about 8–10 disclosed bits. Whatever remains is caught by the 32-bit
verification tag, which aborts the session instead of leaving Alice and Bob
with different keys.

### Privacy amplification with the LHL ceiling

The output is capped at `n_raw − disclosed bits − security_param`. Sessions
without enough entropy are **rejected** instead of producing a seemingly valid
key; below 256 bits, expansion to AES uses HKDF and is declared (HKDF creates no
entropy). The Toeplitz matrix is generated from a fresh seed in every session,
which is announced and signed.

Experimental consequence: an AES-256 key with the full guarantee needs ≥256
extractable bits, i.e. roughly 400 sifted bits. Hence 2000–3000 pulses per
session instead of a few hundred.

### MITM verified in both directions

The MITM block is not inferred from the happy path ("the legitimate signature
is valid") but from the actual rejection of both strategies available to Eve:
reusing Alice's signature on a different message, and signing with her own
keys. `verify_signature_soundness()` runs in Phase 0 and checks that the
signature rejects tampered messages and wrong public keys.

### CHSH with its uncertainty

Each CHSH correlation is estimated on about 1/9 of the pairs, so
`σ_S ≈ 0.15–0.2` with ~800 pairs. S is always reported as `S ± σ`; point
estimates above 2√2 are statistical fluctuations. Below 30 samples per
correlation S is declared not estimable and the CHSH test fails.

---

## Reference results

NetSquid 1.1.8, liboqs 0.16.0, Python 3.11, seed 42, 2000 pulses per session
(3000 for the end-to-end pipeline).

| Scenario (30 km, 4% noise) | QBER | Sifted | Key after PA | Outcome |
|---|---|---|---|---|
| BB84 | 1.63% | 367 | 213 bit | secure |
| BB84 + Eve | 23.48% | 362 | 0 | aborted |
| SSP | 2.22% | 270 | 120 bit | secure |
| SSP + Eve | 33.07% | 254 | 0 | aborted |
| E91 (S = 2.92 ± 0.15) | 3.66% | 191 | 34 bit | secure |

- 20 runs at 30 km: QBER 2.12% ± 0.67% (BB84), 2.06% ± 0.75% (SSP); 40/40
  sessions secure.
- Simulated QBER matches `noise/2` within 0.6 percentage points over 0–20% noise.
- Cascade 6 vs 2 passes: residual-error sessions 13% → 0%, +12.3 disclosed bits.
- Critical interception rate: η* = 39.2% (BB84), 33.9% (SSP).
- Decoy states: R_LMC / R_ideal from 0.57 (10 km) to 0.43 (200 km).
- MITM without authentication: 383 bits of Bob's key known to Eve, apparent
  QBER 1.31%; blocked by ML-DSA-65.

---

## Known limitations

- **QBER estimated on the whole sifted key** rather than on a sacrificed sample,
  and a fixed security parameter of 64 bits **without finite-key analysis**:
  with keys of a few hundred bits, finite-size effects would reduce the
  extractable length further.
- **Additive QBER with a 50% clamp.** Contributions are summed (valid for small
  QBER); beyond 50% the value is clamped, and the `qber_saturated` flag marks
  where the additive model has lost validity.
- **Decoy states analytic and asymptotic**, not fed back into the
  privacy-amplification length.
- **E91 has no eavesdropper model**, and its device independence holds only in
  principle: the Bell violation measured here is not loophole-free.
- **MITM quantum phase in closed form**, not simulated in NetSquid.
- **Shared lattice assumptions** between signature and KEM; signing keys are
  generated per session, whereas a real deployment would use long-term keys
  distributed through an authenticated channel.
- **Idealised timing** for the key rate: 1 GHz, no detector dead time.

---

## Troubleshooting

**`Read-only file system` / `EROFS` under WSL.** Not a code error: Linux has
remounted the WSL disk read-only after finding file-system corruption, typically
after a crash or forced shutdown of Windows. Stop working, then from PowerShell
(as administrator):

```powershell
wsl --shutdown
wsl --mount --vhd "<path to ext4.vhdx>" --bare
wsl --system -u root          # then: lsblk, blkid, e2fsck -f -y /dev/sdX
wsl --unmount "<path to ext4.vhdx>"
```

The path of `ext4.vhdx` is listed by
`Get-ChildItem HKCU:\Software\Microsoft\Windows\CurrentVersion\Lxss | Get-ItemProperty | Select DistributionName, BasePath`.
Keep the code under git and export WSL periodically
(`wsl --export Ubuntu D:\backup\ubuntu.tar`) so a corrupted disk costs minutes,
not work.

**`TypeError: ufunc 'bitwise_and' not supported`.** You are running a v2.1
pre-release with NumPy 1.x; update to the current notebook.

**The kernel does not appear in VS Code.** Open the folder from WSL, make sure
the Jupyter extension is installed *in WSL*, then *Select Kernel → Python
Environments → venv_netsquid* (or *Enter interpreter path* with
`venv_netsquid/bin/python`).

---

## What changed in v2.1

The full list is in the last cell of the notebook. In short:

- notebook split into documented cells, one per module and one per phase;
- Cascade with complete back-propagation (v2.0 updated only one copy of the
  permuted key, so propagation almost never triggered), short-key block caps,
  6 passes;
- key verification with a 32-bit tag;
- K2 of the hybrid key taken **after** privacy amplification;
- random Toeplitz seed per session (v2.0 used a fixed seed and a Hankel matrix);
- all four classical messages signed (v2.0 signed one meaningless byte);
- E91 with the same channel noise as BB84 (v2.0 applied noise twice and ×1.5),
  CHSH reported with σ;
- SLH-DSA recognised under its liboqs ≥ 0.14 name, so the benchmark compares
  both schemes;
- labels, thresholds and conclusions derived from the data instead of
  hard-coded;
- portable output folder, NumPy 1.x compatibility.

---

## References

- Bennett, C. H., & Brassard, G. (1984). *Quantum cryptography: Public key
  distribution and coin tossing.* Proc. IEEE Int. Conf. on Computers, Systems and
  Signal Processing, 175–179.
- Ekert, A. K. (1991). *Quantum cryptography based on Bell's theorem.* Physical
  Review Letters, 67(6), 661–663.
- Bruß, D. (1998). *Optimal eavesdropping in quantum cryptography with six
  states.* Physical Review Letters, 81, 3018–3021.
- Scarani, V., et al. (2009). *The security of practical quantum key
  distribution.* Reviews of Modern Physics, 81, 1301–1350.
- Brassard, G., & Salvail, L. (1994). *Secret-key reconciliation by public
  discussion.* EUROCRYPT '93, LNCS 765, 410–423.
- Hwang, W.-Y. (2003). *Quantum key distribution with high loss: Toward global
  secure communication.* Physical Review Letters, 91, 057901.
- Lo, H.-K., Ma, X., & Chen, K. (2005). *Decoy state quantum key distribution.*
  Physical Review Letters, 94, 230504.
- Ma, X., Qi, B., Zhao, Y., & Lo, H.-K. (2005). *Practical decoy state for quantum
  key distribution.* Physical Review A, 72, 012326.
- Clauser, J. F., Horne, M. A., Shimony, A., & Holt, R. A. (1969). *Proposed
  experiment to test local hidden-variable theories.* Physical Review Letters,
  23(15), 880–884.
- Hadfield, R. H. (2009). *Single-photon detectors for optical quantum information
  applications.* Nature Photonics, 3, 696–705.
- Wegman, M. N., & Carter, J. L. (1981). *New hash functions and their use in
  authentication and set equality.* J. Comput. Syst. Sci., 22(3), 265–279.
- Tomamichel, M., Lim, C. C. W., Gisin, N., & Renner, R. (2012). *Tight
  finite-key analysis for quantum cryptography.* Nature Communications, 3, 634.
- NIST FIPS 203 (ML-KEM), FIPS 204 (ML-DSA), FIPS 205 (SLH-DSA), 2024.
- Krawczyk, H., & Eronen, P. (2010). *HKDF: HMAC-based Extract-and-Expand Key
  Derivation Function.* RFC 5869.
- Coopmans, T., et al. (2021). *NetSquid, a NETwork Simulator for QUantum
  Information using Discrete events.* Communications Physics, 4, 164.

---

**Author:** Francesco Pisaturo — ORCID [0009-0002-5963-8036](https://orcid.org/0009-0002-5963-8036)
