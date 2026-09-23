# Runbook — Distributed Inference on xv6 (Milestone 1)

This runbook reproduces all three required configurations from a clean checkout. Every command below has been run and verified on WSL2 Ubuntu.

**Repository:** <https://github.com/Moorani/xv6-distributed-inference>

---

## Step 0: Clone the Repository

    cd ~
    mkdir -p projects
    cd projects
    git clone https://github.com/Moorani/xv6-distributed-inference.git Milestone-1-
    cd Milestone-1-
    chmod +x fetch-models.sh boot.sh boot-gdb.sh

The rest of this runbook assumes you are inside ~/projects/Milestone-1-.

---

## Prerequisites

**Platform:** Ubuntu (native, or WSL2 on Windows). All commands assume a bash shell.

**System packages:**

    sudo apt update
    sudo apt install -y git build-essential gdb-multiarch qemu-system-riscv \
      gcc-riscv64-linux-gnu binutils-riscv64-linux-gnu binutils-riscv64-unknown-elf

Two binutils packages are installed deliberately — the toolchain produces commands prefixed riscv64-linux-gnu-* (used throughout this runbook) and riscv64-unknown-elf-* (an alternate prefix referenced in some course material). Only one is strictly required for this repo; installing both avoids prefix-mismatch errors.

On some Ubuntu versions the QEMU package is named qemu-system-misc instead of qemu-system-riscv. Either provides the qemu-system-riscv64 binary this project needs.

**Verify the toolchain:**

    qemu-system-riscv64 --version
    riscv64-linux-gnu-objdump --version
    riscv64-unknown-elf-objdump --version
    gdb-multiarch --version

All four should print version info.

**Python environment**, from the repo root:

    python3 -m venv .venv
    source .venv/bin/activate
    pip install scapy pytest coloredlogs

.venv must be re-activated (source .venv/bin/activate) in every new terminal session before running any Python script in this repo.

**Model checkpoints** (not stored in git, ~495 MB total):

    ./fetch-models.sh

Verify:

    ls -la server/models/

Expect: tokenizer.bin (433,869 bytes), stories15M.bin (60,816,028 bytes), stories110M.bin (438,381,596 bytes).

The script re-downloads only what is missing, so it is safe to re-run. The --check flag re-verifies without downloading.

---

## Configuration 1: Single Node

**Terminal 1** — weight server:

    cd ~/projects/Milestone-1-
    source .venv/bin/activate
    python server/server.py

Leave running. Do not close this window.

**Terminal 2** — boot and run:

    cd ~/projects/Milestone-1-
    source .venv/bin/activate
    ./boot.sh

At the xv6 shell prompt ($):

    llama -t 0 -n 32 -i "Once upon a time"

**Expected output:** first run sits at "Fetching from server..." for roughly 80–90 seconds (cold weight fetch over emulated UDP, ~60 MB), then generates coherent text and prints a function-profiling / performance summary block automatically.

**To exit QEMU:** press Ctrl-a, release both keys, then press x alone.

For baseline measurements (Section 8 of the report) specifically, prefix with XV6_CPUS=1:

    XV6_CPUS=1 ./boot.sh

The default (./boot.sh) uses 3 CPUs; the handout fixes the measurement CPU count at 1. Always pass -t 0 (greedy/deterministic) for every measured run — otherwise the default temperature (1.0) samples randomly and no two runs match.

---

## Configuration 2: Multi-Node Ring

Single command boots the whole ring — no separate server.py needed, the simulator starts its own weight server internally.

    cd ~/projects/Milestone-1-
    source .venv/bin/activate
    pkill -f qemu-system-riscv64
    pkill -f mcast_switch.py
    python3 server/distinf_sim.py --workers 2 --model 1 --steps 32 \
      --prompt "Once upon a time" --smp 2 --shard-timeout 1800

**Expected output:** worker registration lines, "layers assigned", per-worker shard fetch confirmation (~40–50 s each on modest hardware), then "all shards resident -- generating", generated text, and a "result PASS" summary line with latency/throughput breakdown.

--model 1 is stories15M.bin (6 transformer layers, supports up to 6 workers); --model 3 is stories110M.bin (12 layers, up to 12 workers).

For 3 workers, change --workers 2 to --workers 3.

To capture the ring transcript to a file for the report appendix:

    script -c "bash" ring_transcript.txt
    # then inside the recording:
    source .venv/bin/activate
    python3 server/distinf_sim.py --workers 2 --model 1 --steps 32 \
      --prompt "Once upon a time" --smp 2 --shard-timeout 1800
    # when it prints PASS, exit the recording:
    exit

---

## Configuration 3: Weight-Fetch Path

**Terminal 1** — weight server (same as Configuration 1):

    cd ~/projects/Milestone-1-
    source .venv/bin/activate
    python server/server.py

**Terminal 2:**

    cd ~/projects/Milestone-1-
    source .venv/bin/activate
    pkill -f qemu-system-riscv64
    ./boot.sh

At the xv6 shell prompt:

    testftp

**Expected output:** "PASS Successfully fetched tokenizer: 433869 bytes", then "PASS Successfully fetched weights: 60816028 bytes", each with a SHA-256 checksum line, ending in "PASS LLM file transfer test completed".

To capture the transcript:

    script -c "./boot.sh" weightfetch_transcript.txt
    # inside the xv6 shell:
    testftp
    # exit QEMU with Ctrl-a, x — the recording closes automatically

---

## Reading the Compiled Kernel (no source shipped)

The kernel ships as a prebuilt binary with a symbol table but no .c source. Use nm and objdump to read it:

    cd xv6-riscv
    riscv64-linux-gnu-nm kernel/kernel | grep -E ' [tT] '
    riscv64-linux-gnu-objdump -d --disassemble=<function_name> kernel/kernel

Example — disassemble a specific function by name:

    riscv64-linux-gnu-objdump -d --disassemble=net_rx kernel/kernel
    riscv64-linux-gnu-objdump -d --disassemble=sys_exec kernel/kernel

The symbol table is also available as a plain text file at kernel/kernel.sym.

---

## Debugging the Kernel Live

Two terminals are needed: one runs the kernel halted waiting for a debugger, the other attaches gdb to it.

**Terminal 1:**

    cd ~/projects/Milestone-1-
    pkill -f qemu-system-riscv64
    ./boot-gdb.sh

The kernel will print its boot banner and then halt, waiting for gdb on port 26000.

**Terminal 2:**

    cd ~/projects/Milestone-1-/xv6-riscv
    gdb-multiarch kernel/kernel

Inside gdb, type each line separately — do not paste multiple gdb commands as a block:

    target remote localhost:26000
    break sys_exec
    continue
    backtrace
    info registers a0 a1 a2
    info registers sp
    stepi
    stepi
    continue

To exit cleanly: Ctrl+C in gdb to interrupt, then quit (confirm with y). Then in Terminal 1, exit QEMU with Ctrl-a, x.

To capture both sides of a session to files:

    # Terminal 1:
    script -c "./boot-gdb.sh" gdb_trace_boot.txt

    # Terminal 2 (after boot has halted):
    script -c "gdb-multiarch kernel/kernel" gdb_trace_session.txt

---

## Known Gotchas

**1. distinf_sim.py silently fails if .venv isn't active.**

The simulator launches its own helper process (mcast_switch.py, the host-side bridge for the emulated network segment) using sys.executable — whatever Python interpreter ran distinf_sim.py itself. If you invoke it with system python3 instead of the venv's python3, the helper crashes on "import coloredlogs" (only installed in .venv) before it can start — but its output is discarded, so the failure is invisible.

The symptom looks unrelated: the master node registers workers fine, then times out trying to fetch the model checkpoint from the host, reporting "arp_lookup: resolution timed out" — because nothing is listening on the host side to answer it.

Fix: always source .venv/bin/activate immediately before running distinf_sim.py.

---

**2. Default --shard-timeout (900s) may not be enough on modest hardware.**

Each simulated node defaults to -smp 8 (8 virtual CPUs); running 3 nodes concurrently oversubscribes machines with fewer than ~24 real threads. Passing --smp 2 reduces contention (ring inference is sequential, so nodes are mostly idle anyway), but the weight/shard fetch itself can still take 40–50+ seconds per worker under load — pass --shard-timeout 1800 to avoid the simulator killing the run as a false-positive timeout mid-fetch.

Symptom without the flag: "FAIL -- never assigned layers within 900s" in the run summary, even though worker registration succeeded.

Fix: add --shard-timeout 1800 to the distinf_sim.py command. On very constrained hardware, you may need to go higher still.

---

**3. Two pkill patterns are needed to fully clean up between runs.**

Not just one:

    pkill -f qemu-system-riscv64
    pkill -f mcast_switch.py

The helper (mcast_switch.py) does not automatically die if a run is interrupted (Ctrl+C) rather than allowed to reach its own cleanup code. Leftover processes on the shared network segment will cause the next run to look like an address collision.

Verify all cleaned up before starting a new run:

    ps aux | grep -E "qemu|mcast"

Only the grep line itself should appear.

---

**4. Ctrl-a, x only works when the terminal running QEMU has focus.**

This is a QEMU monitor escape sequence, not a standard Linux shell shortcut. Press Ctrl and a together, release both keys, then press x alone.

If it doesn't respond — sometimes Windows Terminal intercepts Ctrl-a first — force-kill from a separate terminal instead:

    pkill -f qemu-system-riscv64

---

**5. First llama run is slow; subsequent runs in the same boot are fast.**

The first llama invocation in any boot fetches the 60 MB checkpoint over the emulated UDP network before it can generate anything. Expect 80–90 seconds of "Fetching from server..." on that first run. Once cached in shared memory, subsequent llama commands in the same boot start immediately.

---

**6. Do not run make inside this repo's xv6-riscv/ directory.**

The kernel ships as a prebuilt binary on purpose — its .c source is deliberately withheld (Section 2.9 of the handout). Running make there will fail or do nothing useful. make qemu is only for the separate, plain mit-pdos/xv6-riscv clone from the Hands-On Exercise.

---

## Quick Reference Card

| Task | Command |
|------|---------|
| Activate Python env | source .venv/bin/activate |
| Single-node boot (default 3 CPUs) | ./boot.sh |
| Single-node boot (measurement, 1 CPU) | XV6_CPUS=1 ./boot.sh |
| Ring (2 workers) | python3 server/distinf_sim.py --workers 2 --model 1 --steps 32 --prompt "Once upon a time" --smp 2 --shard-timeout 1800 |
| Ring (3 workers) | same, with --workers 3 |
| Weight-fetch test | (inside xv6) testftp |
| Kernel disassembly | riscv64-linux-gnu-objdump -d --disassemble=<fn> kernel/kernel |
| Clean up between runs | pkill -f qemu-system-riscv64 && pkill -f mcast_switch.py |
| Exit QEMU | Ctrl-a (release), then x |
