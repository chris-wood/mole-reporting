# MOLE Anchor-Bound Quality Reporting

This repository collects experimental drafts and proof-of-concept code for
counting origin-scoped quality outcomes by the Anchor that endorsed a client,
without revealing that Anchor in an individual credential presentation or DAP
report. The profile uses a two-aggregator **Field128** Prio3 variant, with the
Moderator acting as DAP Leader and Collector and a separate Helper holding the
other share. This is a research prototype, not an extension implemented by
ordinary MOLE, Prio3Histogram, or DAP.

## How the binding works

1. An IHAT redemption proof binds the endorsed Anchor at private index `i` to
   a fresh commitment to `j = i + 1`. An ACT issuance proof puts the **same
   hidden `j`** in the credential signed by the Moderator.
2. At presentation the client makes a new commitment
   `C_report = j*H + rho_report*R`. The ACT proof uses the signed `j` in that
   commitment and binds a digest of the **exact DAP report**, its task and
   report ID, and the authenticated origin outcome scope. The client submits
   the presentation, outcome receipt, and report together.
3. The report contains a private one-hot histogram measurement and tag `t`.
   The experimental Prio3 circuit checks `t = selected bucket + 1`. Each
   aggregator computes `P_m = int(t_m)*H + rho_m*R` from its own VDAF tag and
   blinding shares. Both require `P_0 + P_1` to equal `C_report` or
   `C_report + p*H`, where `p` is the Field128 modulus. The second case
   accounts for a carry when canonical field shares are added as integers.
   Only the histogram vector is aggregated.

For one-round Prio3, the Leader sends `P_0` with its verifier share in the
existing aggregation request. The Helper returns `P_1` with the verifier
message in the existing response: **no extra aggregator round trip**. These
point fields, the public commitment and private blinding-share extensions,
and the authenticated assertion of an accepted MOLE presentation are proposed
integration work, not current DAP behavior. Both aggregators must honestly
run the point check for integrity; privacy requires a non-colluding Helper.
The field-carry comparison can disclose a carry bit to the aggregators.

## Documents and code

The four directories below are submodules pinned to the experiment's branches.

| Path | What it contains |
| --- | --- |
| [`mole-drafts`](mole-drafts/draft-chris-mole-anchor-quality.md) | The experimental MOLE profile, the IHAT/ACT Python proof variants, and the cross-protocol tests. |
| [`draft-vdaf-commitment-bound-histogram`](draft-vdaf-commitment-bound-histogram/draft-chris-vdaf-commitment-bound-histogram.md) | The commitment-bound Prio3Histogram proposal. |
| [`draft-irtf-cfrg-vdaf`](draft-irtf-cfrg-vdaf/misc/commitment-bound-histogram.md) | The experimental Python Prio3 validity circuit and VDAF tests, alongside the base VDAF draft. |
| [`libprio-rs`](libprio-rs/src/flp/types/commitment_bound_histogram.rs) | The experimental Rust FLP, Prio3 variant, and P-256 commitment check. |

The local [message-by-message flow](quality-binding-flow.pdf) has
[TeX source](quality-binding-flow.tex); the [one-page explanation of secret
shares and the field carry](anchor-commitment-shares.pdf) has
[TeX source](anchor-commitment-shares.tex). The [sequence diagram](quality-report-sequence.png)
is generated from [Mermaid source](quality-report-sequence.mmd).

## Run the reference experiment

Initialize all nested submodules, including MOLE's proof dependency:

```sh
git clone --branch feat/mole-anchor-reporting --recurse-submodules https://github.com/chris-wood/mole-reporting.git
cd mole-reporting
# For an existing checkout:
git submodule update --init --recursive
```

### End-to-end reporting flow

Install `uv` and Python **3.12 or newer**. From the `mole-reporting` root, run
the one test that exercises the complete reference flow:

```sh
uv run --python 3.12 pytest mole-drafts/poc/tests/test_quality_flow.py -q
```

Expected result: `1 passed`. The test in
[`tests/test_quality_flow.py`](mole-drafts/poc/tests/test_quality_flow.py)
endorses Anchor 1 in a three-Anchor set, redeems it, issues and presents an
ACT credential, verifies the Leader and Helper VDAF shares against the fresh
report commitment, and asserts that the aggregate histogram is `[0, 1, 0]`.

The root [`pyproject.toml`](pyproject.toml) installs the **checked-out
submodules** `mole-drafts/poc` and `draft-irtf-cfrg-vdaf/poc` as editable
local packages. Their code comes from this clone; there is no manual
`PYTHONPATH` setting or separate repository to prepare. The nested MOLE
Sigma Protocols submodule must be initialized as shown above.

### Additional checks

From the same repository root, run all seven focused reporting tests, or the
full MOLE Python suite:

```sh
uv run --python 3.12 pytest -q
uv run --python 3.12 pytest mole-drafts/poc/tests -q
```

With Rust and Cargo installed, run the focused Rust tests from this root too:

```sh
cargo test --manifest-path libprio-rs/Cargo.toml --features experimental commitment_bound
cargo test --manifest-path libprio-rs/Cargo.toml --features experimental wrap_and_wrong_bucket
cargo fmt --manifest-path libprio-rs/Cargo.toml --check
```

The reference tests model the Leader/Helper verification order but do not
provide DAP HPKE transport, wire extensions, or a running aggregation service.
Formal composition, fail-closed wire processing, and replay/result-recording
semantics remain to be specified before interoperable deployment.
