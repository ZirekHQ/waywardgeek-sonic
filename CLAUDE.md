This file lives only on `develop` (this fork's default branch) — never on the topic branches used as upstream PR heads, so it doesn't show up in upstream PR diffs. See "Branch layout" below before cutting a new branch.

# Sonic

C89 library for changing speech speed/pitch/rate without artifacts, via a Hann-windowed sinc-FIR interpolator. This fork (`ZirekHQ/waywardgeek-sonic`) tracks `waywardgeek/sonic`, the canonical upstream.

## Build & test

- `make` — builds the `sonic` CLI, `sonic_lite`, `sonic_experimental`, `libsonic.a`/`.so`, `libsonic_internal.a`/`.so`.
- `make test` — builds and runs `sonic_unit_test` (`tests/runtests.c` + `tests/*_test.c`).
- `make CC=clang CFLAGS="-Wall -Werror -g -fsanitize=address,undefined -fno-omit-frame-pointer" test` — sanitized run; must stay clean (no ASan/UBSan findings, no warnings).
- `make fuzz` — builds `fuzz_sonic` (libFuzzer harness, `tests/fuzz_main.c`). Run bounded: `./fuzz_sonic -max_total_time=120`.
- `make coverage` — builds and runs `sonic_coverage`, emits `.gcov` via `gcov`.
- `.github/workflows/ci.yml` runs all four as separate jobs on push/PR to `master`.

## Key files

- `sonic.c` / `sonic.h` — the library. `adjustRate` → `interpolate` (sinc-FIR resample) drives rate change; `changeSpeed`/`insertPitchPeriod` do pitch-synchronous overlap-add for speed change without pitch change.
- `tests/` — `runtests.c` (driver), one `*_test.c` per concern, `fuzz_main.c` (libFuzzer entry), `genwave.c` (waveform generator used by tests).
- `main.c` / `wave.c` — the `sonic` CLI (WAV in/out).

## Bug patterns already found and fixed here (ASan/UBSan-driven)

- Never left-shift a value that can be negative (`sincTable` has negative entries) — use multiplication instead.
- Accumulate FIR/overlap-add sums in `long`, not `int`; check the widened sum against `INT_MAX`/`SHRT_MIN`/`SHRT_MAX` before narrowing, rather than inferring overflow from a post-hoc sign flip.
- Clamp float sample input to `[-1, 1]` before scaling to `short` — `sonic.h` documents the contract but nothing enforced it.
- `tests/fuzz_main.c`'s `maxSamples` argument to `sonicReadShortFromStream` is a **per-channel** count — cap read requests by dividing the buffer size by `numChannels`, not the raw buffer size.

## Branch layout

- Remotes (local working tree): `wayward-fork` (this repo), `wayward-upstream` (`waywardgeek/sonic`), plus `espeak-fork`/`espeak-upstream` for the related `espeak-ng/sonic` fork worked from the same tree.
- `wayward-main` — local read-only mirror of `waywardgeek/sonic:master`. Never commit to it directly.
- **Upstream PR branches are cut from `wayward-main`, never from `develop`.** This keeps PR diffs minimal and keeps fork-only files (like this one) and unrelated in-flight branches out of what the upstream maintainer reviews.
- `develop` — this fork's default branch, so its own CI runs on push. Not referenced by any open upstream PR.
- `espeak-ng/sonic` has diverged from `waywardgeek/sonic` (different code around some shared functions, different test/build layout). A fix proven here isn't automatically portable — confirm the target function is byte-identical before porting, and adapt Makefile/test wiring to that fork's actual structure.
