# Dataset: Path Recovery Based on End-to-End Quality Prediction and Link-Level Fault Identification in Multi-Hop Networks

Jae-Won Jang, Samia Zahid, Minwoo Jung, Dae-Young Kim
Soonchunhyang University / Kyungpook National University

This repository contains the measurement data from which the experimental results of the manuscript were computed. The testbed is a 14-node Raspberry Pi mesh running babeld, with link degradation applied by Linux netem. Each experiment consists of cycles; a cycle is one run of a scenario on one configuration and is the unit of analysis.

## Contents

```
README.md
cycles.csv                 one row per cycle in this dataset (385 cycles)
data/
  training_e2e.csv         collection experiment, end-to-end measurements, 1 s interval
  training_hop.csv         collection experiment, per-link measurements with labels, 1 s interval
  training_split.json      train / validation / evaluation split of the collection cycles
  per_cycle.csv            recovery experiments, one row per cycle (235 cycles)
  timeseries.csv           recovery experiments, end-to-end measurements and module outputs, 1 s interval
```

## Experiments

| experiment | configuration | scenario | cycles |
|---|---|---|---|
| collection | proposed scheme with recovery disabled (training data) | BASELINE 40, LOCAL 50, SOURCE 40, DELAY_ONLY 10, SINGLE_LINK 10 | 150 |
| comparison | babel: protocol only | LOCAL 20, SOURCE 20 | 40 |
| comparison | naive: threshold-based trigger | LOCAL 20, SOURCE 20 | 40 |
| comparison | ts: time-series prediction only | LOCAL 20, SOURCE 19 | 39 |
| comparison | babel_tuned: protocol only, shortened hello interval | LOCAL 10, SOURCE 10 | 20 |
| proposed | ml: proposed scheme | LOCAL 20, SOURCE 20, DELAY_ONLY 8, SINGLE_LINK 8 | 56 |
| fixed_wide | wide: fixed path-wide recovery | LOCAL 15, SOURCE 15 | 30 |
| loss_only | loss_only: proposed scheme with recovery disabled, loss applied without delay (`arm` = ml in the data) | LOSS_ONLY 10 | 10 |

Scenario names: LOCAL = local degradation, SOURCE = path-wide degradation, DELAY_ONLY = delay without loss, SINGLE_LINK = one link only, LOSS_ONLY = loss without delay, BASELINE = no degradation.

The collection experiment was run with recovery disabled to collect training data. Its 150 cycles are divided by `training_split.json` into train (101), validation (25), evaluation-only (20: DELAY_ONLY and SINGLE_LINK) and excluded (4 local cycles in which degradation was not injected after a path re-check). The excluded cycles are present in the training files and are marked `gate_aborted` in `training_hop.csv`.

The time-series-only configuration has 39 cycles: in one SOURCE cycle the degradation was not applied because the path re-check immediately before injection differed from the gate path, and the cycle was left out of the analysis.

The fixed_wide experiment comprises cycles c03 to c32; the loss_only experiment comprises cycles c02 to c11. Cycle c03_ml-LOSS_ONLY was run twice; this dataset contains the second run.

## cycles.csv

| column | meaning |
|---|---|
| experiment | collection, comparison, proposed, fixed_wide, loss_only |
| cycle_id | cycle name, `c<number>_<arm>-<scenario>`; unique within an experiment |
| configuration | see the table above |
| scenario | see above |
| split | collection cycles only: train, val, eval_only, excluded_gate_abort (the four excluded cycles) |
| t_ramp_start | Unix time (s) at which the degradation ramp started; empty for BASELINE cycles and for the four excluded collection cycles. Subtracting this value from `timestamp` in `timeseries.csv` gives time relative to degradation onset. |

## data/training_e2e.csv

One row per second of the collection experiment, end-to-end from the source to the destination.

| column | meaning |
|---|---|
| cycle_tag | cycle name |
| scenario | scenario |
| k | second index within the cycle |
| t_cycle | seconds since the start of the cycle |
| t_rel_phase, phase | seconds since the start of the current phase, and the phase name |
| flow_warmup | True for the first 3 s after the bulk flow starts; these samples are excluded from aggregates |
| stage | degradation stage derived from `s_eff` |
| s, s_eff | progress of the degradation waveform (0 = not started, 1 = final value); `s_eff` accounts for the starting offset of the link |
| rtt_ms, jitter_ms, retransmits, throughput_mbps | measured round-trip time, jitter, TCP retransmissions and throughput |

## data/training_hop.csv

One row per second per monitored link of the collection experiment. The `label` column is the target of the identification module.

| column | meaning |
|---|---|
| cycle_tag, scenario | as above |
| node, iface, neigh | reporting node, its interface, and the neighbour reached through that interface |
| k, t_rel, stage, s, s_eff | as above, for this link |
| d_m | modelled distance (m) used by the degradation waveform for this link |
| is_target_link | True if degradation was applied to this link in this cycle |
| delay_us_now | delay (microseconds) configured on the link at this second, computed from the injection plan; empty when no delay was configured on the link |
| p_now_pct | the `p` parameter (percent) of the netem Gilbert-Elliott loss model (`loss gemodel`) configured on the link at this second, computed from the injection plan; empty when the link is not a target link |
| label | GOOD or BAD. BAD when the link is a target link and `p_now_pct` at this second exceeds 0.034 % (for DELAY_ONLY, when the configured delay exceeds 6500 µs); GOOD otherwise |
| rtt_ms, jitter_ms, dropped, backlog_pkts, backlog_bytes, throughput_mbps, drop_rate | link measurements reported by the node |
| gate_aborted | True in the four excluded cycles |

## data/per_cycle.csv

One row per cycle of the recovery experiments. The first two columns (`experiment`, `configuration`) identify the experiment and configuration as listed above; the remaining columns are the values recorded by the experiment controller. Columns that exist in only some experiments are empty in the others.

Times are seconds relative to degradation onset unless the name says otherwise. Throughput is in Mbps, RTT and delay in ms unless the name says µs.

| column | meaning |
|---|---|
| cycle | cycle name |
| scenario, arm, profile | scenario, configuration as recorded (`babel` for both babel and babel_tuned), traffic profile |
| path_fwd, path_rev, symmetric | relay nodes of the forward and reverse path at the start of the cycle, and whether they coincide |
| inject_nodes | nodes to which degradation was applied |
| decided_nodes | nodes identified as faulty by the identification stage |
| hit | number of identified nodes that are injected nodes / number of injected nodes |
| t_trigger_ramp | time of the prediction trigger |
| pre_ramp_trigger | True if the trigger fired before degradation onset |
| t_sub10_ramp | first sample below 10 Mbps |
| lead_s | t_sub10_ramp − t_trigger_ramp |
| t_repair_ramp, repair_action | time of the recovery event and the action taken |
| react_s | t_repair_ramp − t_trigger_ramp |
| pre_ramp_repair | True if the recovery event occurred before degradation onset |
| post_tp, post_rtt | mean throughput and RTT in the 10 s after the recovery event |
| ramp_win_tp, ramp_win_rtt | mean throughput and RTT in the window 30-60 s after degradation onset |
| ramp_stall_s | stall time of the bulk TCP transfer |
| blackhole_s, blackhole_conn_s | time during which the destination was unreachable: `blackhole_s` derived from the bulk flow (until throughput is above 0.5 Mbps in two consecutive samples), `blackhole_conn_s` from a new-connection probe (connect() once per second) |
| video_gap_max_s, video_owd_p95_ms, video_loss_rate, video_deadline_over_sent | real-time stream: maximum reception gap, 95th-percentile one-way delay, loss rate, share of packets over the 150 ms deadline |
| n_perhop_rows, perhop_bytes, n_ctrl_msgs | link-state report rows, bytes, and control messages in the cycle |
| brain_cpu_p95, brain_rss_max_mb, predict_ms_p50, classify_ms_p50 | CPU and memory at the source node, median inference time of the two modules |
| gate_ok, base_gate_ok | pre-cycle path check passed |
| dur_s | cycle duration |
| status | execution status of the cycle; OK means the cycle ran to completion. Whether a cycle is part of the analysis is given by `cycles.csv`, not by this column. |
| path_after | path from the source to the destination observed by traceroute after the recovery step of the cycle, including the destination; empty in cycles without a recovery step |

Other columns are diagnostic fields of the experiment controller and are kept as recorded.

## data/timeseries.csv

One row per second of the recovery experiments, end-to-end from the source to the destination, together with the outputs of the prediction and identification modules at that second. The first two columns (`experiment`, `cycle_id`) identify the experiment and the cycle. The `cycle` column holds the controller's internal cycle counter and is not the cycle name; use `cycle_id`.

| column | meaning |
|---|---|
| timestamp | UTC wall-clock time of the sample |
| t_rel | seconds since the start of the cycle |
| phase, state | controller phase and state |
| throughput_mbps, rtt_ms, jitter_ms, retransmits | measurements |
| pred_p10_t1 … pred_p90_t3 | predicted throughput 1, 2 and 3 s ahead: 10th percentile, median, 90th percentile |
| pred_uncertainty_t1 … t3 | prediction uncertainty at each horizon |
| triggered, trigger_count | trigger state at this sample |
| repair_action | recovery action issued at this sample, if any |
| hop_node, hop_iface, classification, confidence, prob_bad | output of the identification module for one link, recorded only in the seconds in which the module ran; empty otherwise. `hop_node` is the address of the reporting node on the experiment mesh (form 10.a.b.2) and `hop_iface` its interface name, as reported by the node |

Rows before the first prediction have empty prediction columns.

## Software

babeld 1.12.1 on Raspberry Pi 4 Model B; Linux netem for loss and delay emulation; PyTorch 2.9.0 (CPU) and LightGBM 4.6.0 for the prediction and identification modules.

Contact: Dae-Young Kim, dyoung.kim@sch.ac.kr

## License

This dataset and its accompanying documentation are licensed under the [Creative Commons Attribution 4.0 International (CC BY 4.0) license](https://creativecommons.org/licenses/by/4.0/). See [LICENSE](LICENSE) for the full terms.

You may share and adapt the material, including for commercial purposes, provided that you give appropriate credit to the authors, link to the license, and indicate any changes.
