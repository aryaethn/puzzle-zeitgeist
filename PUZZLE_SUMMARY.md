# Puzzle Zeitgeist - Complete Analysis Summary

## 📋 Quick Overview

**Puzzle Name:** Zeitgeist  
**Type:** ZK Proof System Vulnerability  
**Difficulty:** Medium-High  
**System:** Halo2 with Poseidon Hash  
**Goal:** Extract Bob's secret private key from public proof data

---

## 🎯 The Challenge

### Story
Bob registered for an airdrop years ago by submitting `commitment = Poseidon(secret, 0)`.  
Now he's trying to claim it by proving he knows the secret, using a nullifier system:
- `nullifier = Poseidon(secret, nonce)` where nonce is random

Bob's proofs keep getting rejected. He suspects SuperCoolAirdrop™ might be malicious and trying to steal his private key.

### Your Mission
You have intercepted 64 of Bob's failed proof attempts. Each contains:
- A ZK proof (Halo2/SHPLONK)
- A nullifier: `Poseidon(secret, nonce_i)`  
- A commitment: `Poseidon(secret, 0)`

**Find Bob's secret!**

---

## 🔍 Technical Details

### Circuit Structure

The Halo2 circuit proves:
```rust
commitment = Poseidon(witness, 0)     // Public output [1]
nullifier  = Poseidon(witness, nonce) // Public output [0]
```

**Public Inputs:**
- `public[0]` = nullifier
- `public[1]` = commitment

**Private Inputs (Witness):**
- `witness` = Bob's secret
- `nonce` = Random value

### Cryptographic Components

1. **Halo2:** Zero-knowledge proof system (scroll-tech fork v1.1.0)
2. **Poseidon Hash:** `P128Pow5T3` variant
   - Width: 3
   - Rate: 2
   - Uses sponge construction
3. **Curve:** BN256
4. **Commitment Scheme:** KZG with SHPLONK

### File Structure

```
puzzle-zeitgeist/
├── commitments/        # 64 files: commitment_0.bin to commitment_63.bin
├── nullifiers/         # 64 files: nullifier_0.bin to nullifier_63.bin  
├── proofs/            # 64 files: proof_0.bin to proof_63.bin
├── halo2/             # Halo2 proving system library
├── poseidon-circuit/  # Poseidon hash implementation
└── src/main.rs        # Main puzzle code
```

---

## 📊 Data Analysis Results

### What We Know (from running the puzzle):

```
✅ All 64 commitments are IDENTICAL
   Value: 0x1f2b5aac96a2527339892fca32dd80b2eb4132b5f90acf27861cba5ddd6ddacf

✅ All 64 nullifiers are UNIQUE
   Total: 64, Unique: 64

❌ Secret is NOT 0
   Poseidon(0, 0) = 0x2ba00861b8f1581f5e17d438e323fa2809f58f1a60009dcd05edb1c9c7c833da
   Actual commitment = 0x1f2b5aac96a2527339892fca32dd80b2eb4132b5f90acf27861cba5ddd6ddacf
```

### Sample Nullifiers:
```
nullifier_0: 0x1e7b658c47c8006f615fa60156ce8556907001993eda2cb0b387b59d38634722
nullifier_1: 0x09ad7f6cf1d37f9a53a39e5cf6e5c506631096d8d900965a47992cbedc1d6ba3
nullifier_2: 0x245819e8dd9535b64540bfceedd5aded3826bb89eb095c8913539b1cd6002bd8
nullifier_3: 0x12f9439f024e80bbce8abc3d6455e7638447f7c6ec4097d55086ab3203ca6de1
nullifier_4: 0x205b951ef59b8857df2185cfc4bb2628380f62fa5ad67f961b5f494bbadbabd8
```

---

## 🎲 Attack Strategy Hypotheses

### Theory 1: Small Secret Space (Most Likely)
**Hypothesis:** The secret is a small number or timestamp.

**Evidence:**
- Puzzle name "Zeitgeist" = "spirit of the times" (German)
- Suggests time/timestamp relevance
- Common CTF pattern: secrets are small or meaningful

**Attack Approach:**
1. Brute force small values: 1-10,000
2. Try timestamps:
   - Unix timestamps around puzzle creation (2021-2024)
   - Date formats: YYYYMMDD, YYMMDD
   - Block numbers
3. Try special values:
   - Common years: 2021, 2022, 2023, 2024
   - Powers of 2
   - Fibonacci numbers

### Theory 2: Algebraic Attack on Poseidon
**Hypothesis:** With 64 hash outputs using the same secret, extract secret algebraically.

**Challenges:**
- Poseidon is designed to be algebraically secure
- Would need vulnerability in this specific parameterization
- More advanced cryptanalysis required

**Attack Approach:**
- Research known Poseidon vulnerabilities
- Try interpolation attacks
- Analyze the polynomial structure

### Theory 3: Proof Information Leakage
**Hypothesis:** Halo2 proofs leak information about the witness.

**What to check:**
- Proof size variations
- Bit patterns in proof bytes
- Polynomial commitment structure
- Known Halo2 vulnerabilities

### Theory 4: Nonce Correlation
**Hypothesis:** Nonces aren't truly random and leak information.

**What to check:**
- Are nonces sequential?
- Related to secret somehow?
- Time-based patterns?

---

## 🛠️ Attack Implementation Plan

### Phase 1: Brute Force Small Secrets
```rust
for candidate in 0..10_000_000 {
    let test_commitment = Poseidon([Fr::from(candidate), Fr::from(0)]);
    if test_commitment == actual_commitment {
        println!("FOUND! Secret = {}", candidate);
        break;
    }
}
```

### Phase 2: Timestamp Bruteforce
```rust
// Try Unix timestamps from 2020-2025
for timestamp in 1577836800..1735689600 {
    // Test timestamp...
}

// Try YYYYMMDD dates
for year in 2020..=2024 {
    for month in 1..=12 {
        for day in 1..=31 {
            let date = year * 10000 + month * 100 + day;
            // Test date...
        }
    }
}
```

### Phase 3: Advanced Analysis
- Analyze proof bytes for patterns
- Check for polynomial relationships
- Research Poseidon security literature

---

## 🔑 Expected Solution Format

Once the secret is found, you need to:
1. Identify which proof index uses that secret (probably all of them)
2. Submit via the TypeForm links in README
3. Provide a write-up explaining the vulnerability

---

## 📚 References

- [Halo2 Documentation](https://github.com/scroll-tech/halo2)
- [Poseidon Hash Specification](https://eprint.iacr.org/2019/458.pdf)
- [SHPLONK Paper](https://eprint.iacr.org/2020/081)

---

## 🚀 How to Run

```bash
cd /Users/aria/Desktop/ZKHack-Puzzles/puzzle-zeitgeist
cargo run --release
```

This will:
1. Display the puzzle description
2. Load all 64 proofs, nullifiers, and commitments
3. Run your attack code
4. Output results

---

## 💡 Key Insights

1. **All commitments identical** → All proofs use the same secret ✅
2. **All nullifiers unique** → Each proof uses different nonce ✅  
3. **64 samples available** → Multiple chances to crack it
4. **Puzzle name hints time** → Consider timestamp-based secrets
5. **ZK proofs are public** → Everything needed to solve is here

---

## ⚠️ Warning

This is a puzzle/CTF challenge. The vulnerability is intentional for educational purposes. Real-world ZK systems using Halo2 and Poseidon are secure when properly implemented.

---

## 📝 Notes

- The puzzle was created for ZK Hack V
- SuperCoolAirdrop™ is fictional
- The vulnerability teaches about side-channels and proof system security
- Focus is on understanding what information proofs expose

---

**Good luck finding Bob's secret! 🔍🔐**

