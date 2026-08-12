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

Re-run `cd ~/btcrecover && source venv/bin/activate` in every new terminal.

## Verify the install

```bash
printf 'TestingOneTwoThre\nTesting%%[OP]neTwoThree\n' > verify.txt
python btcrecover.py \
  --bip38-enc-privkey 6PRVWUbkzzsbcVac2qwfssoUJAN1Xhrg6bNk8J7Nzm5H7kxEbn2Nh2ZoGg \
  --passwordlist verify.txt --has-wildcards --no-eta
```

Must print `Password found: 'TestingOneTwoThree'` in ~2 seconds. Then `rm verify.txt`.

## Run

`passwords.txt` — line 1 is the 10-char version, line 2 is the 12-char version
with `%[A-Z0-9]` at each unknown position. Substitute your real characters:

```
ABDEFGHJKL
AB%[A-Z0-9]DEFGH%[A-Z0-9]JKL
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

### Case unknown, positions known (rows 1–4)

```bash
python btcrecover.py \
  --bip38-enc-privkey 6PR................................................... \
  --passwordlist passwords.txt --has-wildcards \
  --typos 3 --typos-case \
  --no-eta
```

`--typos N --typos-case` allows up to N of the characters to be the other case.
Raise N to 10 for the full 39-hour version. `--typos-capslock` is separate and
cheap — it flips the whole password at once, catching a stuck caps lock key.

### Positions unknown (rows 5–6)

`tokens.txt` holds the 10 known characters on one line:

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

- `--max-adjacent-inserts 2` is required — without it the two unknown characters
  are never placed next to each other and a valid password is missed.
- Drop `--min-typos 2` to also cover 10- and 11-char passwords.
- Add case relaxation on top (row 6): `--typos 12 --max-typos-insert 2
  --typos-case --typos-insert '%[A-Za-z0-9]'`. Run it with `--autosave` and leave
  it going for weeks, or narrow it down first.
- One of the 10 "known" chars is wrong outright: add `--typos 3 --typos-replace
  '%[A-Z0-9]' --max-typos-replace 1`. ~12 days.

## Running it

- `--threads 4` to cap CPU. Default is all cores.
- Ctrl-C prints `Interrupted after finishing password # N`. Resume with the same
  command plus `--skip N`.
- With `--autosave recovery.save`: resume via `python btcrecover.py --restore
  recovery.save`. Tokenlist only, not passwordlist.
- Benchmark your machine: `python btcrecover.py --bip38-enc-privkey 6PR... --performance`
- Don't bother with `--enable-opencl`. Needs ≥6GB VRAM, roughly CPU-parity.

## After it's found

1. Decrypt the `6PR…` key **offline** — local copy of bitaddress.org, Wallet
   Details tab.
2. Confirm the address starts with `1Ld` and matches exactly.
3. Sweep to a new wallet. The old key has been in plaintext on a general-purpose PC.

## Safety

- Run offline. BTCRecover needs no internet.
- Never paste the `6PR…` key or the password into a website, chat, forum, or AI chatbot.
- The key lands in `~/.bash_history`. Clear it: `history -c && rm -f ~/.bash_history`
- Delete `passwords.txt`, `tokens.txt`, `recovery.save` afterwards.

## Reference

- Wildcards and tokenlist syntax: [docs/tokenlist_file.md](docs/tokenlist_file.md)
- Typos options: [docs/TUTORIAL.md](docs/TUTORIAL.md)
- Other coins (`--bip38-currency litecoin|dash|…`): [docs/Usage_Examples/basic_password_recoveries.md](docs/Usage_Examples/basic_password_recoveries.md)
- Upstream: <https://github.com/3rdIteration/btcrecover> (GPLv2)
