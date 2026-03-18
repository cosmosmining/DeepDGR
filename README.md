# DeepDGR Ablation Study — Run Guide

Server: `moura-rtx` (CMU ECE, 8× GPU)

```
ssh chiungct@moura-rtx.ece.cmu.edu
cd /home/chiungct/Differentiable-Global-Router
```

---

## What it runs

54 experiments = 6 benchmarks × 9 (supervised epochs × E2E iterations) combos.

| Supervised epochs | E2E iterations |
|---|---|
| 500, 2000, 5000 | 500, 2000, 5000 |

Benchmarks: `ispd18_test5/8/10_metal5`, `ispd19_test7/8/9_metal5`

---

## Quick start

### Clean previous results

```bash
# Delete all ablation outputs (keeps graph + teacher files)
rm -f ablation_*

# Verify nothing left
ls ablation_* 2>/dev/null && echo "Still have files" || echo "Clean"
```

### Selective clean

```bash
# Re-run one benchmark only
rm -f ablation_ispd18_test10_metal5_*

# Re-run one combo across all benchmarks
rm -f ablation_*_sup5000_e2e5000_*

# Re-run only E2E + DGR steps (keep supervised GNN models)
rm -f ablation_*_e2e_gnn.pth ablation_*_e2e_warmstart.npz ablation_*_e2e_dgr_result.npz ablation_*_e2e_log.csv
```

---

## Method 1: tmux (recommended)

tmux keeps the session alive even if SSH disconnects. You can re-attach anytime.

### First time setup

```bash
# Start a new tmux session named "ablation"
tmux new -s ablation

# Now you're inside tmux — run the script directly:
cd /home/chiungct/Differentiable-Global-Router
rm -f ablation_*
python3 -u run_ablation_e2e.py 2>&1 | tee run_ablation_e2e.log
```

### Detach (leave it running)

Press `Ctrl+B` then `D` — you're back at the normal shell. The script keeps running inside tmux.

### Re-attach (check progress)

```bash
# From any SSH session:
tmux attach -t ablation
```

### Other tmux commands

```bash
# List sessions
tmux ls

# Kill the session (stops the script)
tmux kill-session -t ablation

# Scroll up inside tmux: Ctrl+B then [ then use arrow keys
# Exit scroll mode: press q
```

### Split pane for monitoring

Inside tmux:

```
Ctrl+B then %          # split vertically
Ctrl+B then arrow      # switch panes
```

In the second pane, monitor:

```bash
tail -f run_ablation_e2e.log
```

---

## Method 2: nohup background (no tmux)

### Run

```bash
cd /home/chiungct/Differentiable-Global-Router
rm -f ablation_*
nohup stdbuf -oL -eL python3 -u run_ablation_e2e.py > run_ablation_e2e.log 2>&1 &
echo "PID: $!"
```

What each flag does:

| Flag | Purpose |
|---|---|
| `nohup` | Keeps running after SSH disconnect |
| `python3 -u` | Unbuffered Python output (every print flushes immediately) |
| `stdbuf -oL -eL` | Line-buffered subprocess output (captures DGR solver progress) |
| `> ... 2>&1` | Both stdout and stderr go to the log file |
| `&` | Runs in background |

### Monitor

```bash
# Live output
tail -f run_ablation_e2e.log

# How many combos started / finished
grep -c "COMBO:" run_ablation_e2e.log
grep -c "done" run_ablation_e2e.log

# Any failures
grep "FAILED" run_ablation_e2e.log

# Check if still running
ps aux | grep run_ablation

# Last few lines
tail -20 run_ablation_e2e.log
```

### With timestamps (optional)

```bash
nohup stdbuf -oL -eL python3 -u run_ablation_e2e.py 2>&1 \
  | while IFS= read -r line; do printf '%s %s\n' "$(date '+%H:%M:%S')" "$line"; done \
  > run_ablation_e2e.log &
echo "PID: $!"
```

### Stop it

```bash
# Find the PID
ps aux | grep run_ablation

# Kill it
kill <PID>

# Or kill all related processes
pkill -f run_ablation_e2e.py
```

---

## Check results

### While running

```bash
# Count completed experiments
ls ablation_*_e2e_dgr_result.npz 2>/dev/null | wc -l
# Expected: 54 when done

# See which benchmarks are done
for b in ispd18_test5 ispd18_test8 ispd18_test10 ispd19_test7 ispd19_test8 ispd19_test9; do
  n=$(ls ablation_${b}_metal5_*_e2e_dgr_result.npz 2>/dev/null | wc -l)
  echo "  $b: $n / 9"
done
```

### After completion

The summary table prints at the end of the log:

```bash
grep -A 200 "ABLATION SUMMARY" run_ablation_e2e.log
```

Output files per combo:

```
ablation_<benchmark>_h<H>_l<L>_sup<N>_e2e<M>_model.pth       # supervised GNN
ablation_<benchmark>_sup<N>_e2e<M>_e2e_gnn.pth                # E2E fine-tuned GNN
ablation_<benchmark>_sup<N>_e2e<M>_e2e_dgr_result.npz         # final DGR result
ablation_<benchmark>_sup<N>_e2e<M>_e2e_log.csv                # E2E training log
```

---

## Troubleshooting

### Script skips everything (0s times)

All output files already exist. Clean and re-run:

```bash
rm -f ablation_*
```

### GPU OOM errors

The script auto-retries on other GPUs. If all 8 GPUs OOM, it falls back to smaller model config (h=16, l=2). Check which GPU has memory:

```bash
nvidia-smi
```

### SSH disconnected mid-run

- **tmux**: Just `tmux attach -t ablation` — nothing lost
- **nohup**: Still running, check with `ps aux | grep run_ablation` and `tail -f run_ablation_e2e.log`

### Partial re-run after failure

The script automatically skips completed steps. Just re-run:

```bash
python3 -u run_ablation_e2e.py 2>&1 | tee -a run_ablation_e2e.log
```

Note: `tee -a` appends to the existing log.
