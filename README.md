# BIP38 Password Recovery — Linux Mint

Recovering the password to a `6PR…` BIP38 encrypted private key (address `1Ld…`).

Password: 10 or 12 chars, `A–Z` and `0–9` only. 10 chars known, positions known.

---

## Install

```bash
sudo apt update
sudo apt install -y git python3-venv python3-pip
cd ~
git clone https://github.com/Stoops0311/btcrecover.git
cd btcrecover
python3 -m venv venv
source venv/bin/activate
pip install coincurve pycryptodome ecdsa
```

Re-run this in every new terminal:

```bash
cd ~/btcrecover && source venv/bin/activate
```

## Verify the install

```bash
cd ~/btcrecover
printf 'TestingOneTwoThre\nTesting%%[OP]neTwoThree\n' > verify.txt
python btcrecover.py \
  --bip38-enc-privkey 6PRVWUbkzzsbcVac2qwfssoUJAN1Xhrg6bNk8J7Nzm5H7kxEbn2Nh2ZoGg \
  --passwordlist verify.txt --has-wildcards --no-eta
rm verify.txt
```

Must print `Password found: 'TestingOneTwoThree'` in ~2 seconds.

## Run

Build `passwords.txt` — line 1 is the 10-char version, line 2 is the 12-char
version with `%[A-Z0-9]` at each unknown position. **Replace the example
characters with your real ones:**

```bash
cd ~/btcrecover
cat > passwords.txt <<'EOF'
ABDEFGHJKL
AB%[A-Z0-9]DEFGH%[A-Z0-9]JKL
EOF
```

```bash
python btcrecover.py \
  --bip38-enc-privkey 6PR................................................... \
  --passwordlist passwords.txt \
  --has-wildcards \
  --no-eta
```

1,297 guesses, ~46 seconds. Success prints `Password found: '…'`.

`%[A-Z0-9]` = one character from `A–Z0–9`. `--has-wildcards` enables it in a
passwordlist. Use a passwordlist, not a tokenlist — tokenlist lines get combined
with each other.

## If nothing is found

Relax one assumption at a time, cheapest first. Times assume 28 guesses/sec and
that all 10 known characters are letters — every digit among them halves the
case-related multiplier.

| # | Assumption dropped | Change | Guesses | Time |
|---|---|---|---|---|
| 1 | caps lock was off | add `--typos 1 --typos-capslock` | 2,594 | ~2 min |
| 2 | unknown chars are uppercase | `%[A-Za-z0-9]` in `passwords.txt` | 3,845 | ~2 min |
| 3 | some known chars are the wrong case | 2 + `--typos 3 --typos-case` | 677k | ~7 hr |
| 4 | any known char is the wrong case | 2 + `--typos 10 --typos-case` | 3.9M | ~39 hr |
| 5 | the 2 unknown positions | see below | 85,536 | ~51 min |
| 6 | positions *and* case | 5 + `--typos 12 --max-typos-insert 2 --typos-case` | 260M | ~107 days |
| 7 | the order of the 10 known chars | — | ~10^11 | don't |

### Rows 1–2 — caps lock, lowercase unknowns

```bash
cd ~/btcrecover
cat > passwords.txt <<'EOF'
ABDEFGHJKL
AB%[A-Za-z0-9]DEFGH%[A-Za-z0-9]JKL
EOF

python btcrecover.py \
  --bip38-enc-privkey 6PR................................................... \
  --passwordlist passwords.txt \
  --has-wildcards \
  --typos 1 --typos-capslock \
  --no-eta
```

### Rows 3–4 — case unknown, positions still known

```bash
python btcrecover.py \
  --bip38-enc-privkey 6PR................................................... \
  --passwordlist passwords.txt \
  --has-wildcards \
  --typos 3 --typos-case \
  --no-eta
```

`--typos N --typos-case` allows up to N characters to be the other case. Raise N
to 10 for the full 39-hour version. `--typos-capslock` is separate and cheap — it
flips the whole password at once, catching a stuck caps lock key.

### Rows 5–6 — positions unknown

`tokens.txt` holds your 10 known characters, in order, on one line:

```bash
cd ~/btcrecover
echo 'ABDEFGHJKL' > tokens.txt

python btcrecover.py \
  --bip38-enc-privkey 6PR................................................... \
  --tokenlist tokens.txt \
  --typos 2 --min-typos 2 \
  --typos-insert '%[A-Z0-9]' \
  --max-adjacent-inserts 2 \
  --autosave recovery.save \
  --no-eta
```

- `--max-adjacent-inserts 2` is required — without it the two unknown characters
  are never placed next to each other and a valid password is missed.
- Drop `--min-typos 2` to also cover 10- and 11-char passwords.
- Row 6, positions *and* case — swap the typos flags for:
  `--typos 12 --max-typos-insert 2 --typos-case --typos-insert '%[A-Za-z0-9]'`
- One of the 10 "known" chars is wrong outright: add
  `--typos 3 --typos-replace '%[A-Z0-9]' --max-typos-replace 1`. ~12 days.

## Autosave and resuming

Only needed for the long fallback runs. The main run is 46 seconds — if it gets
interrupted, just run it again.

**`--autosave` works with `--tokenlist` only.** Adding it to a `--passwordlist`
command fails with `error: unrecognized arguments: --autosave`. That's why only
the rows 5–6 command above has it.

### Starting a run with autosave

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

`recovery.save` must not already exist (or must be empty) when you start a *new*
search — if it has data in it, BTCRecover resumes the old run instead of starting
fresh. To start over: `rm recovery.save`.

Progress is written to the file:

- every ~5 minutes while running, and
- immediately when you press Ctrl-C, or the run dies from an out-of-memory error.

So Ctrl-C loses nothing. A power cut loses at most 5 minutes.

### Resuming

Either of these works. The first is simpler:

```bash
cd ~/btcrecover && source venv/bin/activate
python btcrecover.py --restore recovery.save
```

`--restore` must be the **only** option on the command line — every other setting,
including the encrypted key and the tokenlist path, is stored inside the save file.
It prints what it's resuming:

```
Restoring session: --bip38-enc-privkey 6PR... --tokenlist tokens.txt --typos 2 ...
Last session ended having finished password # 1068
Using autosave file 'recovery.save'
```

Or re-run the **exact same command** including `--autosave recovery.save`. It sees
the data in the file and picks up where it left off.

### Rules and failure modes

- **`tokens.txt` must still exist, at the same path, byte-for-byte unchanged.** The
  save file stores a hash of it. Edit it and the resume is refused.
- Changing any option that affects which passwords get generated (`--typos`,
  `--typos-insert`, the key, the tokenlist) makes a resume impossible. You get:
  ```
  ERROR: Can't restore previous session: the command line options have changed
  in a way that will impact password generation.
  ```
  followed by a diff of the offending arguments. Harmless options like `--threads`
  can change freely.
- A save file made by an older version of BTCRecover with different password
  ordering is rejected: `autosave was created with an incompatible version`.
- The file is ~5 KB and holds two save slots, so a crash mid-write can't corrupt
  it — it falls back to the other slot with a warning.

### Resuming without autosave

For `--passwordlist` runs, note the number Ctrl-C prints:

```
Interrupted after finishing password # 683
```

Re-run the *exact same command* plus `--skip 683`.

## Running it

- `--threads 4` to cap CPU usage. Default is every logical core.
- Benchmark your machine:
  ```bash
  python btcrecover.py --bip38-enc-privkey 6PR... --performance
  ```
  Ctrl-C to stop. Divide the search size by the guesses/sec it reports.
- Don't bother with `--enable-opencl`. Needs ≥6GB VRAM, roughly CPU-parity.

## After it's found

1. Decrypt the `6PR…` key **offline** — local copy of bitaddress.org, Wallet
   Details tab.
2. Confirm the address starts with `1Ld` and matches yours exactly.
3. Sweep to a new wallet. The old key has been in plaintext on a general-purpose PC.

## Safety

- Run offline. BTCRecover needs no internet.
- Never paste the `6PR…` key or the password into a website, chat, forum, or AI chatbot.
- The key lands in `~/.bash_history`. Clear it:
  ```bash
  history -c && rm -f ~/.bash_history
  ```
- Delete the working files when done:
  ```bash
  rm -f ~/btcrecover/passwords.txt ~/btcrecover/tokens.txt ~/btcrecover/recovery.save
  ```

## Reference

- Wildcards and tokenlist syntax: [docs/tokenlist_file.md](docs/tokenlist_file.md)
- Typos options: [docs/TUTORIAL.md](docs/TUTORIAL.md)
- Other coins (`--bip38-currency litecoin|dash|…`): [docs/Usage_Examples/basic_password_recoveries.md](docs/Usage_Examples/basic_password_recoveries.md)
