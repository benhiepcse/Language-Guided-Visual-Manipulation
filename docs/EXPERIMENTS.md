# Experiment protocol

Freeze dataset commit/revision, simulator and package versions. Split train/validation by episode; use held-out seeds for online evaluation. Run paired conditions on identical tasks and seeds. Record all episodes including failures, latency, collision count, target selection and final placement error. Report per-task success rate and aggregate; do not equate imitation loss with task success.
