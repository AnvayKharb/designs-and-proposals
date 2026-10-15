# Proposal: Output Contracts and a Streaming Image-Result Pipeline for Printers

- **Status:** Draft
- **Related:** [kubescape/kubescape#3945](https://github.com/kubescape/kubescape/issues/3945) (SARIF combined-scan fix, [#3946](https://github.com/kubescape/kubescape/pull/3946)), [#3670](https://github.com/kubescape/kubescape/pull/3670) (stream JSON writer buffering), [#3794](https://github.com/kubescape/kubescape/pull/3794) (superseded [#3519](https://github.com/kubescape/kubescape/pull/3519)), [#3942](https://github.com/kubescape/kubescape/pull/3942) (Prometheus combined metrics), [#3949](https://github.com/kubescape/kubescape/pull/3949) (PDF combined CVEs), [#4157](https://github.com/kubescape/kubescape/pull/4157) (GitLab SAST mixed-mode rejection), [#2743](https://github.com/kubescape/kubescape/issues/2743) (CSV image-scan rejection), [Proposal #15](partial-scan-outcome-contract.md) (explicit completion and partial-result contracts)
- **Scope:** [kubescape](https://github.com/kubescape/kubescape) CLI: `core/pkg/resultshandling/printer`, `core/core` (image scan orchestration), `cmd/scan`
- **Code read at:** `kubescape/kubescape@60e6fcf` (2026-10-15)
- **Author:** Shivansh Sahu ([@shivansh-gohem](https://github.com/shivansh-gohem) <sahushivansh142@gmail.com>)

## Summary

When a scan produces both posture results and container image results, Kubescape hands them to each output format through one call:

```go
ActionPrint(ctx context.Context, opaSessionObj *cautils.OPASessionObj, imageScanData []cautils.ImageScanData) error
```

Two core architectural problems surround that call:

1. **Silent data loss and ad-hoc error handling across formats.** Each printer historically implements its own branching for "posture + images". Fixes have landed format-by-format (#3946 for SARIF, #3942 for Prometheus, #3949 for PDF, #4157 for GitLab SAST). At least one format (CSV) still silently drops image findings on combined scans, and without a centralized contract test matrix, any new printer or refactoring risks re-introducing silent drops.
2. **Image results are fully materialized in memory.** Every image's `ImageScanData` (including packages, full SBOM, matches, and provider state) is accumulated in a slice from the moment its scan finishes until the process exits. For large clusters with thousands of images, heap consumption scales linearly with image count.

This proposal delivers three concrete solutions:

1. **Explicit, tested combination policy per output format:** Every format declares a policy (`Merge`, `Split`, `Reject`, or `NotApplicable`) enforced by a centralized contract-test matrix across all `printer.AllFormats`.
2. **Close remaining combination gaps:** Provide upfront validation and clear policies for unhandled formats (such as CSV).
3. **An additive streaming image-result pipeline:** Connect an ordered, bounded streaming emitter in `ImageScanOrchestrator` to an additive `IStreamingPrinter` interface, reducing peak memory while maintaining deterministic output order and byte-identical results for existing working scans.

## Motivation

### 1. Per-format fixes keep recurring

The same bug class has required repeated individual patches:

| Fix | Format | Subject |
| :--- | :--- | :--- |
| [#3942](https://github.com/kubescape/kubescape/pull/3942) | Prometheus | emit image vulnerability metrics in combined scans |
| [#3946](https://github.com/kubescape/kubescape/pull/3946) | SARIF | aggregate image runs in combined scans ([#3945](https://github.com/kubescape/kubescape/issues/3945)) |
| [#3949](https://github.com/kubescape/kubescape/pull/3949) | PDF | include image CVEs in combined scan reports |
| [#4157](https://github.com/kubescape/kubescape/pull/4157) | GitLab SAST | reject combined scans to avoid silently dropping image CVEs |

Each was a correct tactical fix, but each printer maintains its own conditional branching. This is also a forward-looking concern: active proposals (such as [attack-path-analysis.md](attack-path-analysis.md)) rely on `IPrinter` and `printer.AllFormats` as the canonical extension points. Without an overarching contract harness, new printers inherit no protection against silent data dropping.

### 2. Format behavior audit on combined scans

Auditing `ActionPrint` across all 16 formats in `printer.AllFormats` at `60e6fcf`:

| Format | Posture + images in one run | Basis |
| :--- | :--- | :--- |
| `pretty`, `json`, `junit`, `prometheus`, `markdown`, `pdf`, `html`, `sarif` | Image data is included (merged, appended, or aggregated) | Code audit |
| `gitlab-sast` | Rejects combined scan upfront with an explicit error ([#4157](https://github.com/kubescape/kubescape/pull/4157)) | Code audit |
| `csv` | `imageScanData` is never read; image findings are silently dropped | Code audit |
| `github-actions` | Ignores `imageScanData`; errors if posture is absent (image findings lack manifest annotations) | Code audit |
| `cyclonedx-json`, `spdx-json` | Image-only by design (encodes container SBOM) | Code audit, `ImageFormats` comment |
| `yaml` | Summarized via the same `buildMachineImageScanSummary` path as JSON | Code audit |
| `policyreport`, `exceptions` | Posture-only by design; ignores `imageScanData` | Code audit |

For CSV, nothing stops the user from running:

```bash
kubescape scan framework nsa ./manifests --scan-images --format csv
```

The scan completes, scans the container images, and writes a CSV file completely lacking image CVE data. While CSV image-only is rejected upfront ([#2743](https://github.com/kubescape/kubescape/issues/2743)), combined scans currently fail silently.

### 3. Memory scaling and history of streaming efforts

The current execution flow (`core/core/scan.go`, `core/core/image_scan.go`):

```text
jobs []ImageScanJob
  │
  ▼
ImageScanOrchestrator.ScanImages
  │ jobChan / resultChan (buffered to len(jobs))
  ▼
results []ImageScanResult  <-- accumulated in full slice
  │
  ▼ sort.Slice by (Image, Platform)  <-- deterministic ordering
ResultsHandler.ImageScanData  <-- appended by value, retained in memory
  │
  ├─► printers: ActionPrint(..., ImageScanData)
  └─► cmd/scan: enforceImageSeverityThresholds(results.ImageScanData, ...)
```

Prior work: [#3794](https://github.com/kubescape/kubescape/pull/3794) superseded [#3519](https://github.com/kubescape/kubescape/pull/3519) for configuration serialization, and [#3670](https://github.com/kubescape/kubescape/pull/3670) streamlined writer buffering for JSON output. (Note: the objection in #3519 was an isolated buffer-pool concurrency issue, not a rejection of streaming architecture.) However, none of these addressed the input-side slice retention in `ImageScanOrchestrator`.

Two principles shape the solution:
1. **Producer must stream:** Changing only the printer signature saves no memory if `ScanImages` continues buffering all results into a slice.
2. **Deterministic output:** Scans currently sort results by `(Image, Platform)`. A streaming design must maintain this ordering without retaining the entire dataset.

## Goals

- Every format declares an explicit, tested combination policy; no combination drops data silently.
- Adding a format to `printer.AllFormats` requires registering its policy (failing the build otherwise).
- Existing scan outputs remain byte-identical.
- Provide an additive streaming path with deterministic ordering and bounded memory footprint.
- Establish reproducible memory benchmarks with defined acceptance criteria.

## Non-goals

- Altering output schemas or fields for existing formats.
- Breaking or removing `IPrinter.ActionPrint` in this phase.
- Streaming within a single image scan (e.g. Syft SBOM generation or Grype vulnerability matching).
- Forcing aggregate formats (HTML, PDF, JUnit, Prometheus) to stream directly; they remain buffered via adapters.

## Proposed Design

### 1. Combination policies (The Contract)

Each format declares its behavior under three scenarios (`PostureOnly`, `ImageOnly`, `Combined`):

| Policy | Meaning |
| :--- | :--- |
| **Merge** | Single artifact contains both posture and image findings |
| **Split** | Image findings are written to a secondary artifact |
| **Reject** | Combination is rejected upfront before scanning begins |
| **NotApplicable** | Format is strictly posture-only or image-only by design |

`ValidatePrinter` in `core/pkg/resultshandling/results.go` enforces this table upfront.

- **`csv, Combined` → `Reject`** (or provide separate sections if approved by maintainers). Rejection matches the #2743 precedent.
- **`gitlab-sast, Combined` → `Reject`** (already implemented via #4157; future work may explore `Split`).
- **`github-actions`, `policyreport`, `exceptions` → `NotApplicable`** for image findings, emitting an explicit user warning rather than silent omission.

### 2. Contract-test matrix

Implemented in `core/pkg/resultshandling/printer/v2/contract_test.go`:

1. **Registry completeness:** Every entry in `printer.AllFormats` must declare a policy for every scenario.
2. **Policy conformance by format type:**
   - **Detailed/Granular formats** (`sarif`, `json`, `gitlab-sast`): verify that every failed control ID and every image vulnerability ID present in the fixture appears in the output document.
   - **Summary/Metric formats** (`prometheus`, summary tables): verify exact aggregate metrics (counts per image, platform, and severity level) rather than individual finding IDs, ensuring Prometheus output remains strictly conformant without schema drift.
3. **Multi-image sibling retention:** Fixtures containing multiple images (including zero-finding images) verify that sibling image findings are never dropped or omitted.
4. **Metamorphic ordering:** Shuffled input jobs produce byte-identical output to verify deterministic ordering.
5. **Schema well-formedness:** Output parses cleanly as valid JSON, XML, or CSV, with SARIF and GitLab validated against local schemas in `testdata/`.

### 3. Streaming interface and Shared Sink Contention

```go
// ImageScanItem represents an incremental result item.
type ImageScanItem struct {
    Data cautils.ImageScanData
    Err  error
}

// IStreamingPrinter allows printers to process image results incrementally.
type IStreamingPrinter interface {
    IPrinter
    ActionPrintStream(ctx context.Context, opaSessionObj *cautils.OPASessionObj, images <-chan ImageScanItem) error
}
```

#### Shared Sink Contention & Replay (Handling `--format json,sarif` to stdout)
When multiple streaming formats target the same unseekable sink (`stdout` without distinct `-o` flags):
- **Concurrent writing to stdout is forbidden:** Interleaving raw JSON and SARIF byte streams corrupts both documents.
- **Sequential streaming requires replay:**
  - *Distinct sinks* (e.g. `--output` specified per format, or a single streaming format to stdout): true streaming pipeline with bounded memory O(window).
  - *Shared unseekable sink* (`stdout` shared by multiple formats): the pipeline activates a **spool-to-disk replay buffer** (NDJSON temporary file). Each image is spooled once as it completes; subsequent printers replay the stream from disk. This preserves bounded Go heap memory without stalling the producer or deadlocking concurrent consumers.

### 4. Producer side: ordered, bounded streaming

`ImageScanOrchestrator.ScanImages` delegates to `ScanImagesStream(ctx, jobs) <-chan ImageScanResult`:
1. Pre-sort jobs by `(Image, Platform)` so target emission order is known upfront.
2. Workers process jobs concurrently.
3. An ordered emitter holds completions in a reorder buffer, releasing result i only after 0..i-1 have been emitted.
4. A bounded semaphore window (proportional to worker concurrency) throttles workers from outpacing the head-of-line emitter, bounding buffer size to O(concurrency) rather than O(images).

### 5. Interaction with Thresholds and Proposal #15

- **Threshold checks:** `enforceImageSeverityThresholds` and the image-only threshold check evaluate a running boolean maximum in-flight as items pass through the pipeline.
- **Alignment with Proposal #15 (Partial Scan Outcome Contract):**
  Proposal #15 defines typed completion states (`Complete`, `Partial`, `Failed`). Under Proposal #15:
  - If a scan finishes with a `Partial` outcome, streaming printers only finalize and flush if the format natively supports partial outcome metadata.
  - On `Failed` or unexpected cancellation, streaming printers cleanly close writers and abort partial document generation to prevent invalid reports from being consumed by CI/CD systems.

### 6. Memory Benchmarks & Acceptance Bar

Extend `core/pkg/resultshandling/printer/v2/streaming_benchmark_test.go`:
- Synthetic `ImageScanData` generators scaling to 100, 1,000, and 5,000 images.
- Measure peak heap, peak RSS, wall-clock time, and output hash equivalence.
- **Acceptance Bar:** On the streaming path with independent sinks, peak heap growth remains flat across 1,000 and 5,000 images, and output remains byte-identical.

## Phased Delivery (12 weeks)

| Weeks | Phase | Deliverables | Exit Criterion |
| :--- | :--- | :--- | :--- |
| **1–2** | **0. Measure & expose** | Complete audit; contract test harness; baseline memory benchmarks | Baseline numbers committed; existing behavior pinned |
| **3–4** | **1. Policy table & matrix** | Centralized policy table in `printresults.go`; `ValidatePrinter` integration; matrix tests | Build fails if format lacks declared policy |
| **5–6** | **2. Close the gaps** | CSV combined rejection (or dedicated section); non-applicable format warnings | No format silently drops findings |
| **7–9** | **3. Streaming pipeline** | `ImageScanItem`, `IStreamingPrinter`, ordered bounded emitter, spool-to-disk shared sink replay | Streamed and batch outputs are byte-identical |
| **10–11** | **4. Format porting** | Port SARIF, JSON, GitLab to streaming; in-flight threshold checks; drop retention on streaming paths | Memory benchmark meets acceptance bar |
| **12** | **5. Handover & Docs** | "Adding a Printer" guide; memory benchmark report; Proposal #15 integration review | Handover documentation complete |

## Risks and Mitigations

| Risk | Mitigation |
| :--- | :--- |
| **Shared stdout interleaving** | Handled via spool-to-disk replay buffer when multiple formats share stdout. |
| **Head-of-line blocking** | Bounded semaphore window limits worker drift; benchmark slow image scenarios. |
| **Byte-level divergence** | Contract matrix tests verify hash equality between streaming and batch outputs. |
| **Goroutine leaks on cancellation** | Strict context propagation and single-producer channel closure rules verified by leak detection tests. |

## Open Questions

1. **CSV Combined Output:** Is upfront `Reject` preferred, or should CSV support multi-table image output?
2. **Streaming API Primitive:** Go channel (`<-chan ImageScanItem`) vs. Go iterator (`iter.Seq[ImageScanItem]`).
3. **Mid-stream failure handling:** Under Proposal #15, should aborted streaming runs delete the output file or write a partial artifact marker?

## What has and has not been verified

- **Read in code (at `60e6fcf`):** The `ActionPrint` bodies of all formats listed; `ValidatePrinter`; `validateCombinedImageScanFlags`; `ScanImages` and `scanImageJobs`; the threshold loop; existing CSV and GitLab tests; the rejection logic introduced in `#4157`.
- **Not yet run by execution:** The CSV combined-scan failure mode (to be covered in Phase 0); live memory benchmarks on image-heavy loads; consumer-side artifact validation for GitLab split artifacts.

## References

- `core/pkg/resultshandling/printer/printresults.go`: `IPrinter`, `AllFormats`, `ImageFormats`
- `core/pkg/resultshandling/printer/v2/*printer*.go`: `ActionPrint` implementations
- `core/core/image_scan.go`: `ImageScanOrchestrator.ScanImages`
- `cmd/scan/scan.go`: `validateCombinedImageScanFlags`
- `core/pkg/resultshandling/printer/v2/streaming_benchmark_test.go`
- `kubescape/kubescape#2743`, `#3670`, `#3794`, `#3942`, `#3945`, `#3946`, `#3949`, `#4157`
- `proposals/attack-path-analysis.md`, `proposals/partial-scan-outcome-contract.md`
