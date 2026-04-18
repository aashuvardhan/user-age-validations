# DNSSEC Toolchain — Programming Assignment 2

A five-part Python toolchain that builds a complete DNSSEC (Domain Name System Security Extensions) validation and analysis suite from scratch, going from basic record validation all the way up to tamper detection and key lifecycle analysis.


---

## Table of Contents


1. [Prerequisites & Installation](#prerequisites--installation)
2. [File Structure & Dependency Chain](#file-structure--dependency-chain)
3. [Question 1 — DNSSEC Validator](#question-1--dnssec-validator)
4. [Question 2 — Recursive Resolver with DNSSEC](#question-2--recursive-resolver-with-dnssec)
5. [Question 3 — Non-Existence Proof (NSEC / NSEC3)](#question-3--non-existence-proof-nsec--nsec3)
6. [Question 4 — Key Lifecycle Analyser](#question-4--key-lifecycle-analyser)
7. [Question 5 — Tamper Detection Demo](#question-5--tamper-detection-demo)


---



## Prerequisites & Installation

### System Requirements

- Python 3.10 or newer (required for the `int | None` and `tuple[str, list]` type-hint syntax used in the code)
- `dig` command-line tool (used in Q5 for before-tampering output)

### Python Packages

Install all required packages with a single command:

```bash
pip install "dnspython[dnssec]"
```

`dnspython[dnssec]` pulls in both `dnspython` (the DNS library) and `cryptography` (for RSA/ECDSA signature verification). Installing them together ensures dnspython can find the cryptography package.

**Why not just `pip install dnspython`?**
Without the `[dnssec]` extra, dnspython installs without the `cryptography` package. When the code calls `dns.dnssec.validate()`, it raises:
```
dns.dnssec.UnsupportedAlgorithm: DNSSEC validation requires python cryptography
```

### Verify the installation

```bash
python3 -c "import dns.dnssec; import cryptography; print('All good')"
```

You should see `All good`. If you see an import error, re-run the pip install above.

### Install `dig` (if missing)

```bash
# Debian / Ubuntu
sudo apt install dnsutils


# macOS (already included with the OS)
```

---

## File Structure & Dependency Chain

```
prog_ass_2/
├── question_1.py   ← Core DNSSEC module (no dependencies on other questions)
├── question_2.py   ← Imports from question_1.py
├── question_3.py   ← Imports from question_1.py
├── question_4.py   ← Imports from question_1.py
└── question_5.py   ← Imports from question_1.py
```

All files must be in the **same directory**. Questions 2–5 use Python's relative import (`from question_1 import ...`), so they will fail if moved elsewhere.

---

## Question 1 — DNSSEC Validator

**File:** `question_1.py`

### What it does

This is the foundation of the whole toolchain. It validates whether a DNS record for a domain is trustworthy by walking the full DNSSEC chain of trust.

The five steps it performs:

| Step | Action | What it proves |
|------|--------|----------------|
| 1 | Fetch the A/AAAA/MX/etc. record | The record exists |
| 2 | Fetch the zone's DNSKEY records | The zone has a public signing key |
| 3 | Fetch the DS record from the parent zone | The parent endorses this zone's key |
| 4 | Verify the RRSIG over the answer using the DNSKEY | The answer wasn't tampered with |
| 5 | Verify the DNSKEY matches the DS digest | The key we used is the one the parent blessed |

All DNS queries go to **Cloudflare (1.1.1.1)** over TCP (UDP is used first with automatic TCP fallback if UDP times out or is blocked by a firewall).

### Key functions exported (used by Q2–Q5)

| Function | Purpose |
|----------|---------|
| `get_answer(domain, rdtype)` | Fetch any record type + its RRSIG |
| `get_dnskey(domain)` | Fetch DNSKEY records + RRSIG |
| `get_ds(domain)` | Fetch DS record from parent zone |
| `validate_rrsig_with_dnskey(rrset, rrsig, dnskey)` | Cryptographically verify a signature |
| `validate_dnskey_with_ds(dnskey_rrset, ds_rrset)` | Check DNSKEY matches parent DS digest |
| `validate_dnssec(domain, rdtype)` | Full five-step validation, returns result dict |
| `print_result(result)` | Pretty-print the result dict |

### Commands

```bash
# Validate a DNSSEC-signed domain
python3 question_1.py example.com A

# Try different record types
python3 question_1.py example.com MX
python3 question_1.py ietf.org AAAA

# Test with an unsigned domain (will show INVALID — no DNSKEY)
python3 question_1.py google.com A
```

---

## Question 2 — Recursive Resolver with DNSSEC

**File:** `question_2.py`
**Imports from:** `question_1.py`

### What it does

Implements a full iterative DNS resolver that walks the DNS tree the same way a real resolver does — starting at the root, following NS referrals down through the TLD, and finally reaching the authoritative server. At every level of delegation it validates DNSSEC using the Q1 functions.

Resolution path:
```
Root (198.41.0.4)  →  .com nameservers  →  example.com nameservers
```

At each hop the resolver:
1. Validates the DNSKEY, RRSIG, and DS for that zone (using Q1)
2. Queries the current nameserver for the target domain
3. Follows the NS referral if the answer isn't there yet
4. Falls back to Cloudflare TCP if a nameserver is unreachable directly

The root zone's trust anchor is skipped (it would require a hardcoded KSK) — validation starts at the TLD level.

### Commands

```bash
# Resolve an A record with DNSSEC validation at each hop
python3 question_2.py example.com A

# Other record types and domains
python3 question_2.py ietf.org AAAA
python3 question_2.py cloudflare.com A

# Unsigned domain — resolves successfully but DNSSEC shows NOT VERIFIED
python3 question_2.py google.com A
```

---

## Question 3 — Non-Existence Proof (NSEC / NSEC3)

**File:** `question_3.py`
**Imports from:** `question_1.py`

### What it does

Handles the case where a domain or record type does not exist in a DNSSEC-signed zone. Instead of just returning "not found", DNSSEC provides a cryptographic **proof of non-existence** using NSEC or NSEC3 records.

Two response types are handled:

| Response | Meaning |
|----------|---------|
| `NXDOMAIN` | The domain name itself does not exist anywhere in the zone |
| `NODATA` | The domain exists but has no record of the queried type |

Two proof record types are supported:

| Type | How it works | Zone walking risk |
|------|-------------|------------------|
| `NSEC` | Lists the next existing name in alphabetical order. Proves qname falls in the gap. | High — attacker can walk all names by following the chain |
| `NSEC3` | Hashes all names with SHA-1 + salt + iterations. Proves the queried hash falls in the gap. | Low — pre-images are computationally hard to reverse |

The validation steps:
1. Send the query with the DNSSEC OK (DO) bit set
2. Classify the response (NXDOMAIN / NODATA / EXISTS)
3. Extract NSEC/NSEC3 records from the authority section
4. Verify the RRSIG over each NSEC/NSEC3 using Q1's `validate_rrsig_with_dnskey()`
5. Confirm the NSEC/NSEC3 actually covers the queried name (range check)

### Commands

```bash
# NODATA — domain exists but no TXT record (NSEC proof from Cloudflare white-lie)
python3 question_3.py mail.example.com TXT

# NXDOMAIN — name doesn't exist (NSEC3 proof)
python3 question_3.py nonexistent.iana.org A

# EXISTS — record found, no non-existence proof needed
python3 question_3.py example.com A

# NODATA at iana.org (NSEC3 zone)
python3 question_3.py iana.org TXT
```

---

## Question 4 — Key Lifecycle Analyser

**File:** `question_4.py`
**Imports from:** `question_1.py`

### What it does

Inspects a live DNSSEC zone and classifies what state its key lifecycle is in. Real zones regularly rotate their signing keys through carefully timed "rollover" procedures, and this tool detects exactly where in that process a zone currently is.

The analysis steps:
1. Fetch all DNSKEY records and split into KSKs (flags=257) and ZSKs (flags=256)
2. Read the DNSKEY RRSIG to find which KSK is currently active
3. Fetch the DS record from the parent zone
4. Compute key tags (RFC 4034 Appendix B algorithm) and match every DNSKEY against the DS
5. Fetch an A record RRSIG to find which ZSK is currently active
6. Classify the overall lifecycle state

### Lifecycle states detected

| Status | Condition |
|--------|-----------|
| **Normal** | Single KSK, single ZSK, DS matches the KSK |
| **ZSK Rollover in Progress** | Two or more ZSKs co-exist, single stable KSK |
| **KSK Rollover in Progress** | Two or more KSKs; DS matches only some of them |
| **Multiple KSKs Active (DS match all)** | All KSKs match DS — multi-signer or duplicate |
| **DS Mismatch** | DS in parent zone doesn't match any published DNSKEY |
| **Not DNSSEC-signed** | No DNSKEY or DS found at all |

The tool also reports:
- RRSIG validity windows (inception date, expiration date, days remaining)
- A warning flag if any signature expires within 3 days

### Commands

```bash
# Normal state
python3 question_4.py cloudflare.com
python3 question_4.py ietf.org

# ZSK rollover (example.com maintains multiple ZSKs)
python3 question_4.py example.com

# Unsigned domain
python3 question_4.py google.com
```


---

## Question 5 — Tamper Detection Demo

**File:** `question_5.py`
**Imports from:** `question_1.py`

### What it does

Demonstrates how DNSSEC detects when DNS records have been modified in transit. Real, legitimately signed records are fetched and then intentionally modified **without re-signing** — simulating what an attacker in the middle might do. The Q1 validator is then run against both the original and the tampered versions.

The demo covers four parts as required by the assignment:

| Part | Description |
|------|-------------|
| **A — Setup** | Fetch real DNSSEC-signed records; show `dig` output with AD flag |
| **B — Tampering** | Create two tampered versions in memory (no actual DNS change) |
| **C — Verification** | Run the custom validator on all three versions |
| **D — Analysis** | Explain what failed, where, and why |

### Two tampering scenarios

**Scenario A — A record IP changed**
The IP address in the A record is replaced with `1.3.3.7` while the original RRSIG is kept unchanged. This is the classic man-in-the-middle attack: redirect traffic to a malicious server without having the private key to re-sign.

**Scenario B — RRSIG signature bytes corrupted**
The first and last bytes of the cryptographic signature are flipped with XOR `0xFF`. The A record data is left completely intact. This simulates a corrupted signature in a cache or in transit.

In both cases validation fails at **Step 4 (RRSIG verification)** — the signature check catches the modification before the chain-of-trust check (Step 5) is even reached.

### Commands

```bash
# Run against example.com (default)
python3 question_5.py

# Run against any DNSSEC-signed domain
python3 question_5.py example.com
python3 question_5.py ietf.org

# The script exits with code 0 if both tampering scenarios are correctly
# detected as INVALID and the baseline is VALID — useful for scripting
echo "Exit code: $?"
```
