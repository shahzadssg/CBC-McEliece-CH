# CBC-McEliece-CH

Reference implementation of the **CBC-McEliece-CH** chameleon hash and the **CBC-MEL-San** post-quantum sanitizable signature scheme it lifts to. This is the artifact accompanying the CANS 2026 paper _"CBC-McEliece-CH: Post-Quantum Sanitizable Signatures from Code-Based Chameleon Hashing via Cipher-Block Chaining"_ (under double-blind review).

The construction iterates a derandomized Classic McEliece primitive in cipher-block-chaining mode, splitting each ciphertext into a length-$k$ left half $L$ and a length-$\ell = n - k$ right half $R$. Both halves feed the next stage, with $R$ extended to $k$ bits by a length-regular pad. Each chained step after the first absorbs two message blocks, halving the McEliece-call count. Sanitization invokes a single Patterson decode regardless of message length.

The repository covers three NIST PQC security levels (L1, L3, L5) via `Classic-McEliece-348864`, `Classic-McEliece-460896`, and `Classic-McEliece-6688128`.

---

## Quick start

### Linux / macOS

```bash
pip install liboqs-python matplotlib numpy
jupyter notebook CBC-McEliece-CH.ipynb
```

`liboqs-python`'s auto-installer builds `liboqs` from source on first import. You need `cmake`, `ninja`, and a C compiler:

```bash
sudo apt install cmake ninja-build build-essential   # Debian/Ubuntu
brew install cmake ninja                              # macOS
```

### Windows (Anaconda)

The `liboqs-python` auto-installer requires a C toolchain that Anaconda environments do not provide by default. The simplest path is to use the pre-built DLL in `windows-liboqs/`:

1. Open `windows-liboqs/` in File Explorer.
2. Double-click `install-liboqs.bat`. It copies `oqs.dll` and `liboqs.dll` (byte-identical copies under both names) to `%USERPROFILE%\_oqs\bin\`.
3. Restart your Jupyter kernel and run the notebook.

See `windows-liboqs/README.md` for build provenance, SHA-256, and manual install instructions.

---

## What's in the notebook

`CBC-McEliece-CH.ipynb` is the main artifact for reviewers. It's organized into 14 numbered sections with markdown explanations between every code cell:

1. **Environment Check** --- detects `liboqs-python` availability, prints diagnostic if missing.
2. **Classic McEliece Parameter Sets** --- the three NIST levels and their parameters ($n$, $k$, $\ell$, $|\text{pk}|$, $|\text{ct}|$).
3. **Cryptographic Primitives** --- SHAKE-256, XOR, the length-regular pad.
4. **PKE Backends** --- `DetPKEStub` (correctness) and `McElieceKEMTimer` (timing).
5. **The Construction** --- `CBC_McEliece_CH` class with `hash`, `verify`, `adapt`.
6. **Correctness Validation** --- Hash → Adapt → Verify round trip at all three NIST levels for $t \in {1, 2, 4, 8, 16, 64}$.
7. **Semantic Tests** --- ADM size constraint, non-rejoin block preservation, missing final-stage rejection.
8. **Encryption Count Verification** --- confirms Hash uses $\lceil(t+1)/2\rceil$ encryptions, Adapt uses one decrypt plus $k - 1$ encrypts.
9. **Edge Cases** --- empty message, wrong block size, $t = 1$, $|\text{ADM}| = 0$.
10. **Timing Methodology** --- 5 warm-up + 100 measured iterations, median + IQR.
11. **Classic McEliece KEM Microbenchmarks** --- real liboqs measurements (skipped cleanly when liboqs is unavailable).
12. **Chain Timing Projections** --- Hash/Adapt cost projected from measured KEM primitives.
13. **Visualization** --- scaling plots at L1/L3/L5 with both Hash and Adapt curves.
14. **Honest Verdict** --- what was validated and what was not.

The correctness sections (1–9) run without `liboqs` using the deterministic PKE stub. The timing sections (11–13) require `liboqs` for real measurements.

---

## Construction overview

For $t$ message blocks $m = (m_1, \ldots, m_t)$ of $k$ bits each and IV of $k$ bits, the chameleon hash chain has $k_{\text{stage}} = \lceil(t+1)/2\rceil$ stages:

- **Stage 1**: $c_1 = \mathsf{E}^{\det}(\mathrm{IV} \oplus m_1)$.
- **Stage** $i \geq 2$: $c_i = \mathsf{E}^{\det}\bigl(L_{i-1} \oplus \mathsf{pad}(R_{i-1}) \oplus m_{j_a} \oplus m_{j_b}\bigr)$ where $j_a = 2i - 2$ and $j_b = 2i - 1$, with $m_{j_b} = 0^k$ if $j_b > t$.

Each ciphertext $c_i$ splits as $c_i = L_i | R_i$ with $|L_i| = k$ and $|R_i| = \ell = n - k$. The final $c_{k_{\text{stage}}}$ is the hash output.

**Adapt** runs the chain forward with a fresh IV* through stages $1, \ldots, k_{\text{stage}} - 1$, then forces a designated "rejoin block" in the final stage to absorb the algebraic constraint enforced by a single Patterson decode of $h$.

The full construction (Construction 1) and Adapt algorithm (Algorithm 1) are formalized in §3 of the paper.

---

## Security claims

`CBC-MEL-San`, the sanitizable signature scheme that lifts `CBC-McEliece-CH` into a full signing scheme, is proven secure in the strong BCDKS/KSS model with eight properties: unforgeability, immutability, privacy, transparency, signer-accountability, sanitizer-accountability, invisibility, and unlinkability. Reductions are to syndrome decoding for binary Goppa codes and EUF-CMA of an outer ML-DSA signature. Proofs are in Appendix B of the paper.

---

## Reproducing the paper's numbers

The paper's measured timings come from `liboqs` 0.15.0 with the AVX2-optimized Classic McEliece variants on a Linux x86_64 workstation. To reproduce closely:

1. Install Ubuntu 22.04 or later on AVX2-capable x86_64 hardware.
2. Build `liboqs` from source with `OQS_DIST_BUILD=ON`. The AVX2 and VEC variants of Classic McEliece are enabled automatically on Linux.
3. `pip install liboqs-python==0.15.0`.
4. Run `python3 benchmark.py --iter 1000` for higher statistical confidence.

Numbers will scale predictably with CPU clock. The encap/decap ratio should remain close to 1:1.5.

---

## Honest limitations

Three things are worth knowing before you draw conclusions from this code.

1. **The chain uses a deterministic PKE stub, not real derandomized Classic McEliece.** `liboqs` exposes only the KEM API (encap/decap), not the underlying derandomized PKE. The stub in `DetPKEStub` uses SHAKE-256 to produce an injective map and lets us validate every structural aspect of the chain (the L/R split, the pad, the rejoin-block algorithm) end to end. For paper-grade timings, we measure `liboqs` encap and decap directly and project the chain cost. Appendix D of the paper documents three paths to a real derandomized PKE (patch `liboqs`, reimplement against PQClean, or switch to deterministic Niederreiter).
    
2. **`liboqs` does not enable AVX2 variants of Classic McEliece on Windows.** The relevant CMake block in `liboqs` is guarded by `CMAKE_SYSTEM_NAME MATCHES "Linux|Darwin"`. A Windows build (native or cross-compiled) includes only the reference C implementation, which is ~3–5× slower than the AVX2-optimized version. The pre-built DLL we ship in `windows-liboqs/` is reference-only and validates the construction but doesn't match the paper's optimized numbers.
    
3. **Benchmark numbers depend on hardware.** Single-thread frequency, AVX2 support, memory bandwidth, and OS scheduling all affect timing. Median of 100 iterations with warm-up is good engineering practice but not the rigor of a dedicated benchmark harness on quiet hardware. Reproduce on your own machine before drawing conclusions about absolute performance.
    

---

## Dependencies

|Component|Version|Purpose|
|---|---|---|
|Python|3.10+|Tested with 3.10–3.12|
|`liboqs-python`|0.15.0|Classic McEliece, ML-KEM, ML-DSA bindings|
|`liboqs`|0.15.0|C shared library (auto-installed on Linux/macOS)|
|`matplotlib`|3.5+|Plots in notebook section 13|
|`numpy`|1.21+|Linear trend fitting in section 13|
|`jupyter`|any recent|Running the notebook|


---

## Building `liboqs` from source

On Linux, the `liboqs-python` auto-installer handles this. To do it manually (recommended if you want explicit control over the variants):

```bash
git clone --depth=1 --branch 0.15.0 https://github.com/open-quantum-safe/liboqs.git
cd liboqs && mkdir build && cd build
cmake -GNinja \
  -DCMAKE_BUILD_TYPE=Release \
  -DOQS_DIST_BUILD=ON \
  -DBUILD_SHARED_LIBS=ON \
  -DOQS_USE_OPENSSL=OFF \
  ..
ninja
sudo ninja install
```

For Windows, see `windows-liboqs/README.md` for the cross-compilation recipe used to produce the included DLL.

---

## Troubleshooting

**`RuntimeError: No oqs shared libraries found`** --- `liboqs-python` cannot find the C library. On Linux/macOS, install `cmake` + `ninja` and re-run the auto-installer. On Windows, use the pre-built DLL in `windows-liboqs/`.

**`liboqs-python` raises `SystemExit: 1`** --- auto-installer failed (no C toolchain). Same fix as above. The notebook catches this exception so the correctness path still runs.

**Slow decap on Windows (decap >> encap)** --- your `liboqs.dll` was built without `-O3`. The DLL in `windows-liboqs/` is built with proper optimization. Replace any existing copy at `%USERPROFILE%\_oqs\bin\liboqs.dll`.

**Notebook plots empty or noisy** --- most likely cause is single-iteration timing. Use the helper in section 10 (`measure_median`) with at least 100 iterations.

---

## Citation

This paper is currently under double-blind review at CANS 2026. Citation information will be added on acceptance. For now:

```bibtex
@misc{cbc-mceliece-ch-2026,
  title  = {CBC-McEliece-CH: Post-Quantum Sanitizable Signatures from
            Code-Based Chameleon Hashing via Cipher-Block Chaining},
  note   = {Under double-blind review at CANS 2026},
  year   = {2026},
}
```

---

## License

To be determined. Suggested: MIT for the implementation, CC BY 4.0 for the paper text. Both are common choices for academic artifacts and impose minimal restrictions on reuse and reproduction.

The Classic McEliece reference implementation and `liboqs` are distributed under their own licenses (MIT for `liboqs`). See those projects' respective `LICENSE` files for details.

---

## Contact

Author contact information will be added on acceptance.
