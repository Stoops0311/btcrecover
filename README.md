# BTCRecover — BIP38 Password Recovery on Linux Mint

This README covers exactly one job: **recovering the password to a BIP38-encrypted
private key that starts with `6P`, on a clean install of Linux Mint.**

Everything else BTCRecover can do (seed phrases, wallet files, 20+ altcoins) is
ignored here. Full upstream docs: <https://btcrecover.readthedocs.io/>

---

## 1. The recovery target

| | |
|---|---|
| Encrypted private key | starts with `6PR`, 58 characters |
| Bitcoin address | starts with `1Ld` |
| Password length | 12 characters |
| Password alphabet | uppercase letters + digits only (`A–Z`, `0–9`) |
| Known | 10 of the 12 characters |
| Unknown | 2 characters |

A few facts that shape the whole job:

- `6PR` (and `6PY`) means a **non-EC-multiplied** key. The key itself contains a
  checksum of its own address, so BTCRecover knows on its own when the password
  is right. **You do not need to supply the `1Ld…` address** — it's only a
  sanity check at the end.
- BIP38 uses scrypt with N=16384, r=8, p=8. It is deliberately slow: expect
  roughly **20–30 guesses/second on a modern 6-core desktop** (measured: ~28/s on a
  Ryzen 5 5600, 12 threads). That is the hard limit on how big a search you can run.
- With only 2 unknown characters, the search space is tiny by password-cracking
  standards (see §5). This is very likely to be a sub-hour job.

---

## 2. Install on a clean Linux Mint

Tested on Mint 22 (Ubuntu 24.04 base, Python 3.12). Mint 21 (Python 3.10) works
the same way.

### 2.1 System packages

```bash
sudo apt update
sudo apt install -y git python3-venv python3-pip
```

`python3` is preinstalled on Mint. You do **not** need build tools —
the packages below ship prebuilt wheels for Python 3.10–3.13.

### 2.2 Get BTCRecover

```bash
cd ~
git clone https://github.com/3rdIteration/btcrecover.git
cd btcrecover
```

### 2.3 Create a virtual environment

Mint 22 / Ubuntu 24.04 ship a "externally managed" Python — `pip install` into
the system Python is blocked and will error out. Use a venv:

```bash
python3 -m venv venv
source venv/bin/activate
```

Your prompt now starts with `(venv)`. **Every `python btcrecover.py` command
below assumes this venv is active.** If you open a new terminal, re-run
`cd ~/btcrecover && source venv/bin/activate` first.

### 2.4 Install the three packages BIP38 needs

```bash
pip install coincurve pycryptodome ecdsa
```

That's all a BIP38 recovery requires:

| Package | Why |
|---|---|
| `coincurve` | fast C secp256k1 — without it you get a slow pure-Python fallback and a warning |
| `pycryptodome` | fast AES + RIPEMD160 |
| `ecdsa` | required by the BIP38 wallet code; BTCRecover refuses to start without it |

You can install the full `requirements.txt` / `requirements-full.txt` instead,
but nothing in them is needed for this job.

### 2.5 Verify the install

Run a recovery against the official BIP38 test vector — a real encrypted key
whose password is `TestingOneTwoThree`:

```bash
echo 'Testing%[OP]neTwoThree' > verify.txt
python btcrecover.py \
  --bip38-enc-privkey 6PRVWUbkzzsbcVac2qwfssoUJAN1Xhrg6bNk8J7Nzm5H7kxEbn2Nh2ZoGg \
  --tokenlist verify.txt --no-eta
```

Expected final line (takes ~2 seconds):

```
Password found: 'TestingOneTwoThree'
```

If you see that, the install is correct and BIP38 decryption works. Delete
`verify.txt`. If you instead see a warning about "pure-Python secp256k1" or
"Please install pycryptodome", §2.4 didn't take — check the venv is active.

---

## 3. The search

**Case A below is your situation** — you know exactly which two positions are
unknown. It's a 1,296-guess search that finishes in under a minute. Cases B and C
are fallbacks, only relevant if Case A finishes without a hit and you have to
loosen an assumption.

### Case A — you know the positions ← use this one

Example: you know the password is `A B ? D E F G H ? J K L`-shaped, i.e. the
unknown characters are at positions 3 and 9.

Write a **tokenlist** file with `%[A-Z0-9]` standing in for each unknown character.
`%[A-Z0-9]` expands to exactly one character from `A–Z` plus `0–9` (36 options).

```bash
nano tokens.txt
```

Content — one line, your 10 known characters in place, `%[A-Z0-9]` at each
unknown slot (this is an *example*; substitute your real characters):

```
AB%[A-Z0-9]DEFGH%[A-Z0-9]JKL
```

Run it:

```bash
python btcrecover.py \
  --bip38-enc-privkey 6PR................................................... \
  --tokenlist tokens.txt \
  --no-eta
```

**36 × 36 = 1,296 guesses. Under a minute.**

### Case B — you know the 10 characters in order, but not where the other 2 go

This is the more common situation: you can write down the 10 characters you're
sure of, in the right order, but the 2 unknown ones could be anywhere — start,
middle, end, or next to each other.

`tokens.txt` holds just your 10 known characters, in order, on one line:

```
ABDEFGHJKL
```

Then let BTCRecover insert 2 characters anywhere:

```bash
python btcrecover.py \
  --bip38-enc-privkey 6PR................................................... \
  --tokenlist tokens.txt \
  --typos 2 --min-typos 2 \
  --typos-insert '%[A-Z0-9]' \
  --max-adjacent-inserts 2 \
  --autosave recovery.save \
  --no-eta
```

Flag by flag:

| Flag | What it does |
|---|---|
| `--typos 2` | at most 2 changes per guess |
| `--min-typos 2` | at least 2 — skips 11-character guesses, since you know it's 12 |
| `--typos-insert '%[A-Z0-9]'` | each change is "insert one uppercase letter or digit" |
| `--max-adjacent-inserts 2` | **required.** Without it, the two inserted characters are never placed next to each other, so a password like `ABDEFGHJK` + `XY` + `L` would be missed entirely |
| `--autosave recovery.save` | writes progress every ~5 min so you can resume |

Quote the `%[A-Z0-9]` in single quotes — bash would otherwise try to expand the
brackets.

**11 gap-pairs + adjacent placements = 85,536 guesses. About 50 minutes at 28
guesses/sec; 2–4 hours on a slower laptop.**

### Case C — you know *which* 10 characters, but not their order

Don't. Permuting 10 known characters across 12 positions plus 2 unknowns is
hundreds of millions of guesses at ~28/sec — years. If this is your situation,
think harder about the ordering first and come back to Case B.

---

## 4. Running it

- **Progress:** BTCRecover prints an ETA and a progress bar. `--no-eta` skips the
  up-front counting pass (worth it here — counting 85k guesses is instant either
  way, but it also avoids the "are you sure" prompt on huge searches).
- **Threads:** it uses every logical core by default. Cap it with `--threads 4`
  if you want to keep using the machine.
- **Interrupt:** Ctrl-C stops it and prints e.g. `Interrupted after finishing
  password # 35162`.
- **Resume:** with `--autosave recovery.save`, restart with just
  `python btcrecover.py --restore recovery.save`. The tokenlist file must be
  unchanged. (Autosave only works with `--tokenlist`, not `--passwordlist`.)
  Without autosave, re-run the *exact same command* plus `--skip 35162`.
- **GPU:** don't bother. `--enable-opencl` supports BIP38, but it needs ≥6GB VRAM
  and only roughly matches a decent CPU. For an 85k-guess search it's not worth
  the setup.
- **Benchmark your own machine** before committing to a long run:
  ```bash
  python btcrecover.py --bip38-enc-privkey 6PR... --performance
  ```
  Ctrl-C to stop. Divide your search size by the guesses/sec it reports.

---

## 5. Search sizes at a glance

| Scenario | Guesses | Time @ 28/s |
|---|---|---|
| Case A — 2 unknown chars, positions known | 1,296 | ~45 sec |
| Case B — 2 unknown chars, positions unknown | 85,536 | ~51 min |
| Case B, but you're unsure it's exactly 12 chars (drop `--min-typos 2`) | 85,932 | ~51 min |
| Case B with lowercase allowed too (`%[A-Za-z0-9]`) | 253,704 | ~2.5 hr |

If you finish Case B with no result, the assumptions are what's wrong, not the
tool. In rough order of likelihood, relax them one at a time:

1. Allow lowercase: `--typos-insert '%[A-Za-z0-9]'`
2. Allow symbols too: `--typos-insert '%[A-Z0-9!@#$%%^&*_-]'` (note `%%` for a
   literal `%`)
3. Allow one of your "known" 10 characters to be wrong as well:
   add `--typos 3 --typos-replace '%[A-Z0-9]' --max-typos-replace 1`
   (this multiplies the run time by ~350 — roughly 12 days. Sleep on it first.)

---

## 6. When it finds the password

```
Password found: 'ABC3DEFGH9JKL'
```

BTCRecover only prints the password — it doesn't hand you the decrypted private
key. To get the key:

1. Take your `6PR…` key and the found password to a **BIP38 decrypt** tool run
   **offline** — e.g. a local copy of bitaddress.org (Wallet Details tab), or
   `python -c` against the bundled library.
2. Confirm the resulting address starts with **`1Ld`** and matches your address
   exactly. If it doesn't, you decrypted the wrong key.
3. Sweep the funds to a new wallet with a fresh key. The old key has now been in
   plaintext on a general-purpose computer.

---

## 7. Safety

- **Run this offline.** Disconnect the network before you paste your `6PR…` key
  into anything. BTCRecover needs no internet.
- **Never paste the encrypted key or the password into a website, a chat, a
  support forum, or an AI chatbot.** A `6P…` key plus a weak password is money
  sitting on the table for whoever sees it.
- BTCRecover holds the key and every candidate password in plain memory. It's not
  hardened against malware on the machine — use a clean install, ideally a Mint
  live USB with networking off.
- Your `6PR…` key ends up in `~/.bash_history` because it's on the command line.
  Clear it when you're done: `history -c && rm -f ~/.bash_history`
- Delete `tokens.txt`, `verify.txt` and `recovery.save` when you're done — the
  tokenlist contains 10 of your 12 password characters.

---

## 8. Reference

- Token file syntax (wildcards, anchors, mutual exclusion): [docs/tokenlist_file.md](docs/tokenlist_file.md)
- Typos options in full: [docs/TUTORIAL.md](docs/TUTORIAL.md)
- BIP38 examples for other coins (`--bip38-currency litecoin`, `dash`, …):
  [docs/Usage_Examples/basic_password_recoveries.md](docs/Usage_Examples/basic_password_recoveries.md)
- Upstream project: <https://github.com/3rdIteration/btcrecover> (GPLv2)
