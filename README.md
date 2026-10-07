# Kratos — Pearl (PRL) GPU miner

A fast int8 tensor-core miner for Pearl, by **Spiry Research** (Spiry Ark SRL).

**1 % dev fee** — half of what SRBMiner and PeakMiner charge, and below tw-pearl's
1.5 %. On a 0 %-fee pool such as HeroMiners you keep **99 %** of what you mine.

---

## Quick start

**One line** (detects your GPU, verifies the checksum, installs to `/usr/local/bin`):

```sh
curl -fsSL https://github.com/SPIRY-RO/kratos-releases/releases/latest/download/install.sh | sh
kratos mine -o stratum+tcp://de.pearl.herominers.com:1200 -u YOUR_PRL1_ADDRESS -w rig01
```

**Tarball** (if you would rather read it first — and you should):

```sh
curl -fsSLO https://github.com/SPIRY-RO/kratos-releases/releases/latest/download/kratos-linux-x86_64.tar.gz
curl -fsSLO https://github.com/SPIRY-RO/kratos-releases/releases/latest/download/SHA256SUMS
sha256sum -c SHA256SUMS --ignore-missing     # verify before you run it
tar xzf kratos-linux-x86_64.tar.gz && sudo install -m755 kratos /usr/local/bin/
```

**Docker** (what most rig and cloud operators want):

```sh
docker run --rm --gpus all ghcr.io/spiry-ro/kratos:latest \
  mine -o stratum+tcp://de.pearl.herominers.com:1200 -u YOUR_PRL1_ADDRESS -w rig01
```

---

## Check your host before you mine on it

This is the part other miners do not give you. Rented and shared GPUs are often not
what they claim, and the difference is large: on an RTX 4090 we have measured
**287 TH/s on a clean host and 148–154 TH/s on a host with a hidden co-tenant** —
the same card, the same binary.

```sh
kratos devices --health
```

```
health: idle sample 1: 382 MiB used, 0% util, 83 W
health: mma peak 355.5 T-MAC/s (expected ~352)
health: fold 286.9 TH/s | sm 2280 MHz mem 10251/10501 MHz | 350/350 W (default 450 W) | clk_reasons 0x4
health: POWER_CAPPED (enforced power limit is below the default) | gpu 0 NVIDIA GeForce RTX 4090 | exit 11
```

| Verdict | Exit | Means |
|---|---|---|
| `OK` | 0 | Clean card at full power. Mine on it. |
| `FAKE_SHARED` | 10 | Something else was already using this GPU before we started. Expect roughly half speed. |
| `POWER_CAPPED` | 11 | The enforced power limit is below the card's default, or the SW power cap is engaged. |
| `MEMCLK_LOCKED` | 11 | Memory clock held below maximum under load. |
| `SLOW_UNKNOWN` | 12 | Slower than this model should be, cause not identified (or NVML unavailable). |

Add `--json` for one machine-readable line, which is what you want in a rig script.

---

## Measured speed

Honest numbers, each with the conditions it was measured under. Pearl speed depends
more on the **host** than on the miner, so a figure without its power limit and
co-tenant state is worthless.

| GPU | Kratos | Conditions |
|---|---|---|
| RTX 4090 | **295.4 TH/s** (844 GH/W) | clean host, enforced **350 W** of a 450 W default |
| RTX 4090 | 286.9 TH/s (820 GH/W) | a second clean host, also enforced 350 W of 450 W |
| RTX 3090 | **118.3 TH/s** (396 GH/W) | clean host, enforced **299 W** of a 390 W default |

On that 3090, Krig 1.5.6 managed 81.7 TH/s (272 GH/W) on the **same card at the same
cap** — Kratos is 44 % faster there, and it gets that by drawing less power per
multiply-accumulate, so the card's own governor returns the saving as clock (1440 MHz
against 1110 MHz).

Every GPU hit is checked: `bench --verify` recomputed 400 of 400 against an independent
CPU implementation of the whole PearlHash chain, across 57 operand draws, and correctly
rejected a deliberately corrupted hit.

Both cards above were **power-throttled by their host**, not run at stock. Per watt
Kratos is roughly 18 % ahead of the published 450 W figures for the closed-source
miners. We publish GH/W alongside TH/s for that reason.

Supported: **sm_86** (RTX 30-series, incl. laptop), **sm_89** (RTX 40-series, incl.
laptop), **sm_120** (RTX 50-series). Needs an NVIDIA driver of 550 or newer.
Pascal (GTX 10-series) has no int8 tensor cores and cannot run Pearl at all.

---

## Usage

```
kratos mine -o URL [-o URL2 ...] -u WALLET [-w WORKER] [--devices all|0,2]
  -o URL         stratum+tcp://de.pearl.herominers.com:1200 (repeatable; the first is
                 primary, and we return to it 10 min after a failover)
  -u WALLET      your prl1… address
  -w WORKER      worker name, default 'kratos'
  --devices      'all' (default) or a comma-separated list of CUDA indices

kratos devices --health [--json]    judge the host before mining on it
kratos bench [--seconds N] [--verify]    local rate; --verify recomputes hits on the CPU
kratos selftest                     known-answer tests of the verifier, no GPU needed
kratos --version                    version and build-id
```

Pools known to work (plain TCP; TLS is not supported yet):

```
stratum+tcp://de.pearl.herominers.com:1200     HeroMiners, 0 % pool fee, pays any prl1 address
stratum+tcp://pearl.luckypool.io:3360          LuckyPool
stratum+tcp://prl.2miners.com:1818             2miners
```

**Always pass more than one `-o`.** A single pool url means no failover, and a stalled
route is not an obvious failure: the connection stays up, the pool simply stops sending
new work, and every hit you find is against a job the network has moved past. We lost
twenty minutes of a rental to exactly that before adding a second region. With two or
more, Kratos moves on after three consecutive failures and returns to the first url ten
minutes later.

```sh
kratos mine -u YOUR_PRL1_ADDRESS -w rig01 \
  -o stratum+tcp://de.pearl.herominers.com:1200 \
  -o stratum+tcp://fi.pearl.herominers.com:1200 \
  -o stratum+tcp://us2.pearl.herominers.com:1200
```

HeroMiners regions: us, us2, ca, br, de, fi, fr, tr, sg, hk, kr, au. Pick the two or
three closest to your rig.

Output is line-oriented and stable, so a supervisor can parse it:

```
Total: 286.9 TH/s shares: 12 accepted 1 stale 0 rejected
gpu0: 286.9 TH/s | NVIDIA GeForce RTX 4090 | sm 2280 MHz mem 10251 MHz | 350/350 W 51 C | clk_reasons 0x4
```

Only the summary line begins with `Total: `; every other line starts with a
lowercase tag (`gpu0:`, `pool:`, `job:`, `share:`, `health:`).

---

## The dev fee

Kratos takes **1 % of mining time**. For 36 seconds of each hour the pool login
changes to Spiry Research's address under the worker name `kratos-devfee`, then
changes back. **The GPU never stops and your hashrate does not change** — only the
stratum login does, which is the same way SRBMiner, PeakMiner and llmjob take
theirs.

The first slice falls at a random point in the first hour, so the rate is 1 %
however long you run, and restarting gains you nothing.

---

## Exit codes

| Code | Means |
|---|---|
| 0 | Clean exit |
| 2 | No usable GPU, or every card was lost |
| 3 | No accepted share for 20 minutes while hashing — the supervisor should restart or move host |
| 4 | A card stopped making progress for 30 s; the process exits because a wedged CUDA context cannot be unwound |
| 5 | The pool refused the login — check your address |
| 10/11/12 | Health verdicts, see above |
| 64 | Bad arguments |

## Troubleshooting

**`pool: authorize refused [20]`** — the wallet is not a valid prl1 address, or the
pool wants `address.worker` in one field. Kratos sends wallet and worker separately,
which is what HeroMiners expects.

**Shares rejected** — run `kratos bench --verify`. It recomputes hits on the CPU
against an independent verifier; `PASS` means the GPU results are sound and the
problem is the pool dialect, not the kernel.

**About half the speed you expected** — run `kratos devices --health`. Nearly every
such report is `FAKE_SHARED` or `POWER_CAPPED`.

**A TLS pool does not connect** — not supported yet. Use the plain-TCP port.

---

## Licence

Proprietary. © Spiry Ark SRL. Binary redistribution and reverse engineering are
prohibited — see `LICENSE.txt`, which also carries the MIT notice for the
open-source portions Kratos includes.

Issues and questions: https://github.com/SPIRY-RO/kratos-releases/issues
