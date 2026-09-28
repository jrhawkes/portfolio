The new tarball, `nvl72-burnin-1.3.tar.gz`, is below. The thresholds now come from NVIDIA documentation where it exists, and the rest are marked as values to calibrate. All 120 tests pass.

**Thresholds that changed**

| Check | 1.1 value | 1.3 value | Basis |
|---|---|---|---|
| STREAM Triad per Grace | ≥ 900 GB/s | ≥ 307 GB/s for 480 GB Grace; ≥ 400 GB/s for 120 GB | NVIDIA's minimum Triad for a Grace-Hopper/Blackwell superchip is 400 GB/s with 120 GB and 307 GB/s with 480 GB, at 72 threads. The agent reads each NUMA node's memory size and picks the matching value. Any other size, such as 240 GB, fails unless you set an explicit threshold. |
| Grace↔Grace (cross-socket) STREAM | none | ≥ 100 GB/s each direction | NVIDIA requires at least 100 GB/s unidirectional, measured with STREAM, for a Grace Blackwell node to be considered healthy. |
| C2C host→GPU (H2D), per GPU | ≥ 800 GB/s | ≥ 300 GB/s | On a healthy 480 GB system, host-to-device bandwidth should be about 350–360 GB/s. The 900 GB/s figure is the link's bidirectional bandwidth, not something a one-way copy can reach. |
| C2C GPU→host (D2H), per GPU | none | ≥ 145 GB/s | On Grace-Hopper/Blackwell x4 systems this should be about 170 GB/s, because more copy engines are reserved for GPU-to-GPU NVLink traffic. |
| NVLink GPU↔GPU copy, worst pair | ≥ 900 GB/s | ≥ 630 GB/s, measured before and after burn-in | The theoretical rate is 1.8 TB/s bidirectional per GPU, so 900 GB/s one way. One GB200 user reported 717 GB/s (~80% of theoretical) on an nvbandwidth copy test. |
| NVLink links per GPU | GPU 0 only | all 18 links active on every GPU, each ≥ 47.5 GB/s | 18 links × 50 GB/s per direction. A link that trained at a lower speed now fails. |
| GPU thermal | junction ≤ 90 °C | T.Limit ≥ 5 °C during burn-in; abort at 0; 90 °C kept as a backstop | Newer NVIDIA drivers report T.Limit, the number of degrees left before the GPU target temperature, instead of fixed temperature limits. |
| Driver and services | ≥ 560; nvidia-fabricmanager | ≥ R570 (optional list of approved versions); nvidia-imex and nvidia-persistenced | NVIDIA lists R570 and R580 drivers for GB200, and GPU driver and IMEX as the compute-tray components. The fabric manager runs on the NVLink switch trays (nmx-controller). |
| Rack power alert | > 210 kW | > 125 kW; < 60 kW during burn; any tray below 80% of the rack median | Rack is about 120 kW. The per-tray figure is my estimate. |

**New checks**
- **Fabric registration.** Every GPU must report fabric State Completed, Status Success, and full bandwidth.
  - All 18 trays must report the same NVLink cluster ID and partition (clique) ID; the orchestrator fails the rack otherwise.
  - This is the only rack-wide NVLink check so far.
- **Peer comparison.** C2C, NVLink copy and STREAM fail if any value is more than 10% below its tray's median. gpu_burn fails if any GPU is more than 5% below.
  - This catches a single weak part without needing a fleet baseline.
- **Thresholds in one place.** Each agent reports every check together with the limit it used, and the report rolls those up across the 18 trays.
  - The report no longer contains any hard-coded thresholds. It was the second copy of the thresholds in 1.1, and it had two checks that always passed.
- **STREAM build.** STREAM is now compiled per NVIDIA's recipe (`-Ofast -mcpu=neoverse-v2`, 120M elements, 200 iterations), so results are comparable to NVIDIA's reference minimums.

**Still needs checking on real trays**
- **C2C and NVLink floors.** NVIDIA notes that bandwidth numbers depend on the specific Grace Hopper/Blackwell SKU, IOMMU settings and GPU clock settings. Calibrate the 300 GB/s, 145 GB/s and 630 GB/s floors on known-good trays.
- **Site-dependent limits.** The idle temperature limits and the Grace 90 °C cap depend on your coolant supply and haven't been changed.
- **Unconfirmed output formats.** I couldn't capture live GB200 output for three things:
  - the `temperature.gpu.tlimit` field name
  - the layout of the Fabric section in `nvidia-smi -q`
  - the nvbandwidth GPU-to-GPU matrix for 4 GPUs

  If the field is missing, the agent falls back to the 90 °C cap. If the Fabric section is missing, boot validation fails.

The question you should have asked but didn't: *is the 2% GPU-throughput regression limit tighter than gpu_burn's normal run-to-run variation?* It may be. Two 15-minute runs taken 72 hours apart can differ by 1–3% from coolant temperature and clock behavior alone. Measure repeated runs on a known-good tray before you trust that limit. Otherwise it will produce false failures, and the new per-GPU peer check is the more reliable signal.

Confidence:
- 90% for the STREAM, cross-socket, IMEX and driver changes (NVIDIA documentation).
- 70% for the C2C floors, which rest on NVIDIA's general guidance, not GB200-specific numbers.
- 60% for the NVLink copy floor (one field report).
- 65% that the T.Limit field and Fabric-section parsing match your driver without adjustment.
