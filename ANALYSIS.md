# Puzzle Zeitgeist - Analysis

## Overview
This is a ZK Hack puzzle involving a vulnerability in an airdrop claiming system that uses zero-knowledge proofs. The puzzle name "Zeitgeist" suggests something about the spirit of the times or capturing a moment in time.

## Puzzle Story
Bob signed up for SuperCoolAirdrop™ years ago by providing a hash (commitment) of his private key. Now he's trying to claim his airdrop by:
1. Proving knowledge of the private key matching his commitment
2. Providing a nonce and nullifier (to prevent replay attacks)

But Bob's proofs keep getting rejected, making him suspicious that SuperCoolAirdrop™ might be trying to steal his private key.

## Technical Architecture

### Cryptographic Components

1. **Halo2 Circuit**: Using the Halo2 proving system (via scroll-tech fork)
2. **Poseidon Hash**: Used for both commitments and nullifiers
   - Width: 3
   - Rate: 2
   - Two separate Poseidon instances in the circuit

### The Circuit (`MyCircuit`)

The circuit has two main computations:

1. **Commitment Calculation**:
   ```
   commitment = Poseidon(witness, 0)
   ```
   - Takes the secret witness (private key)
   - Hashes it with constant 0
   - This is exposed as a public input at position 1

2. **Nullifier Calculation**:
   ```
   nullifier = Poseidon(witness, nonce)
   ```
   - Takes the secret witness
   - Hashes it with a randomly chosen nonce
   - This is exposed as a public input at position 0

### Public Inputs
The circuit has 2 public inputs:
- `public_input[0]`: nullifier
- `public_input[1]`: commitment

### File Structure

```
puzzle-zeitgeist/
├── commitments/        # 64 commitment files (commitment_0.bin to commitment_63.bin)
├── nullifiers/         # 64 nullifier files (nullifier_0.bin to nullifier_63.bin)
├── proofs/            # 64 proof files (proof_0.bin to proof_63.bin)
├── halo2/             # Halo2 proving system library
├── poseidon-circuit/  # Poseidon hash circuit implementation
└── src/
    └── main.rs        # Main puzzle code
```

## The Vulnerability (To Be Discovered)

### Current Attack Code
The skeleton code in `main()` attempts to find which commitment matches `Poseidon(0, 0)`:

```rust
let secret = Fr::from(0u64);
let secret_commitment = poseidon_base::primitives::Hash::<
    _,
    P128Pow5T3Compact<Fr>,
    ConstantLength<2>,
    WIDTH,
    RATE,
>::init()
.hash([secret, Fr::from(0u64)]);

for i in 0..64 {
    let (_, _, commitment) = from_serialized(i);
    assert_eq!(secret_commitment, commitment);
}
```

### Current Output

When running the puzzle:
```
=== ANALYZING DATA ===

Commitments:
  ✓ All 64 commitments are IDENTICAL
  Value: 0x1f2b5aac96a2527339892fca32dd80b2eb4132b5f90acf27861cba5ddd6ddacf

Nullifiers:
  Total: 64
  Unique: 64
  ✓ All nullifiers are unique

First 5 nullifiers:
  nullifier_0: 0x1e7b658c47c8006f615fa60156ce8556907001993eda2cb0b387b59d38634722
  nullifier_1: 0x09ad7f6cf1d37f9a53a39e5cf6e5c506631096d8d900965a47992cbedc1d6ba3
  nullifier_2: 0x245819e8dd9535b64540bfceedd5aded3826bb89eb095c8913539b1cd6002bd8
  nullifier_3: 0x12f9439f024e80bbce8abc3d6455e7638447f7c6ec4097d55086ab3203ca6de1
  nullifier_4: 0x205b951ef59b8857df2185cfc4bb2628380f62fa5ad67f961b5f494bbadbabd8

=== TESTING ATTACK ===

Testing if secret = 0:
  Expected commitment: 0x2ba00861b8f1581f5e17d438e323fa2809f58f1a60009dcd05edb1c9c7c833da
  Actual commitment:   0x1f2b5aac96a2527339892fca32dd80b2eb4132b5f90acf27861cba5ddd6ddacf
  ✗ NO MATCH. Need to find the actual secret.
```

This tells us:
- `Poseidon(0, 0) = 0x2ba00861b8f1581f5e17d438e323fa2809f58f1a60009dcd05edb1c9c7c833da`
- `commitment_0.bin = 0x1f2b5aac96a2527339892fca32dd80b2eb4132b5f90acf27861cba5ddd6ddacf`
- All commitments are identical (expected - same secret for all proofs)
- All nullifiers are unique (expected - different random nonces)

## The Attack Vector

### Key Observations

1. **64 Proofs with Random Nonces**: The puzzle provides 64 different proofs, each with a different random nonce
2. **Same Secret**: All 64 proofs are generated with the same secret witness (Bob's private key)
3. **Public Information**: We can see:
   - All 64 nullifiers (public)
   - All 64 commitments (should all be identical)
   - All 64 proofs

4. **Nullifier Formula**: 
   ```
   nullifier_i = Poseidon(secret, nonce_i)
   commitment = Poseidon(secret, 0)
   ```

### Potential Attack Strategy

The vulnerability likely involves:

1. **Algebraic Relationships**: With multiple Poseidon hash outputs using the same secret but different nonces, there might be a way to extract information about the secret

2. **Small Secret Space**: The secret might be from a limited range, allowing brute force

3. **Nonce Leakage**: The random nonce generation or proof system might leak information

4. **Birthday Paradox**: With 64 proofs, there might be collisions or patterns

5. **Poseidon-Specific Attack**: There might be a known weakness in how Poseidon is parameterized or used

### What We Need to Find

Our goal is to determine Bob's secret private key from the public information (commitments, nullifiers, and proofs).

## Possible Attack Vectors to Investigate

### 1. **Small Secret Space / Brute Force**
The secret might be from a limited range. The puzzle might expect us to:
- Try common small values (1, 2, 3, ... 1000000)
- Check if the secret is a timestamp (puzzle name "Zeitgeist" = spirit of the times)
- Test if it's a date/time when the proofs were generated

### 2. **Algebraic Attack on Poseidon**
With 64 samples of `Poseidon(secret, nonce_i)`, there might be:
- A way to solve for the secret algebraically
- Linear algebra attack if enough equations can be formed
- Interpolation attack on the Poseidon permutation

### 3. **Proof/Circuit Information Leakage**
The Halo2 proofs themselves might leak information:
- Proof size variations
- Constraint system structure
- Public inputs ordering
- Witness compression artifacts

### 4. **Nonce Predictability**
Even though `OsRng` is used, check if:
- Nonces follow a pattern
- They're related to timestamps
- Sequential relationships exist

### 5. **Nullifier Collision Analysis**
With 64 nullifiers:
- Birthday paradox might apply
- Look for near-collisions
- Analyze bit patterns

### 6. **"Zeitgeist" Hint**
The puzzle name suggests time-related elements:
- Secret might be a Unix timestamp
- Could be block number, date in YYYYMMDD format
- Year 2024, 2023, or specific date

## Next Steps

1. ✅ **Verify all commitments are identical** - DONE: All 64 are the same
2. ✅ **Verify all nullifiers are unique** - DONE: All 64 are unique
3. **Try brute-forcing small secret values** (0-10000)
4. **Try timestamp-based values** around when puzzle was created
5. **Analyze the actual proof bytes** for patterns
6. **Research Poseidon algebraic attacks** for this specific configuration

## Tools and Dependencies

- Rust nightly-2024-11-19
- Halo2 (scroll-tech fork v1.1.0)
- Poseidon hash (from scroll-tech)
- BN256 curve (via halo2curves)
- KZG commitment scheme with SHPLONK

## Running the Puzzle

```bash
cargo run --release
```

This will:
1. Load all 64 serialized proofs, nullifiers, and commitments
2. Run the attack code
3. Currently fails at assertion checking if secret=0

