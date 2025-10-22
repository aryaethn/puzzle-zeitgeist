# Puzzle Zeitgeist - Detailed Mechanism Explanation

## 🎓 Understanding the System

This document explains exactly how the puzzle works, what each component does, and why Bob's proofs are failing.

---

## 1️⃣ The Airdrop Registration System

### Phase 1: Registration (Years Ago)

Bob wants to register for SuperCoolAirdrop™. The process:

1. **Bob generates a secret:** Some private value only he knows
2. **Bob creates a commitment:**
   ```
   commitment = Poseidon_Hash(secret, 0)
   ```
3. **Bob submits the commitment:** He sends this to SuperCoolAirdrop™
4. **Commitment is stored:** SuperCoolAirdrop™ remembers Bob's commitment

**Why this design?**
- Bob doesn't reveal his secret
- The commitment "locks in" his secret
- Later, he can prove he knows the secret without revealing it

### Phase 2: Claiming (Today)

Bob receives notification that his airdrop is ready. To claim it:

1. **Bob picks a random nonce:** A fresh random value for this claim
2. **Bob creates a nullifier:**
   ```
   nullifier = Poseidon_Hash(secret, nonce)
   ```
3. **Bob creates a ZK proof:** Proves that:
   - He knows a `secret` such that `commitment = Poseidon(secret, 0)`
   - He computed `nullifier = Poseidon(secret, nonce)` correctly
   - **Without revealing** the secret or nonce
4. **Bob submits:** `(proof, nullifier, commitment)`

**Why the nullifier system?**
- Prevents replay attacks
- Each nullifier can only be used once
- Same secret can generate multiple different nullifiers (different nonces)
- If someone tries to reuse a proof, the nullifier will be detected

---

## 2️⃣ The Zero-Knowledge Proof Circuit

### What the Circuit Proves

```rust
Circuit MyCircuit {
    // Private inputs (witness)
    witness: Fr,  // Bob's secret
    nonce: Fr,    // Random nonce for this proof
    
    // Public inputs
    nullifier: Fr,    // public[0]
    commitment: Fr,   // public[1]
    
    // Constraints
    PROVE that:
        commitment == Poseidon(witness, 0)
        nullifier == Poseidon(witness, nonce)
}
```

### Step-by-Step Circuit Execution

1. **Load Private Inputs:**
   - `witness` (Bob's secret) → Loaded as advice (private)
   - `nonce` → Loaded as advice (private)
   - Constant `0` → Loaded as fixed

2. **First Hash - Commitment:**
   ```
   commitment_chip = Pow5Chip (Poseidon hasher)
   commitment_digest = commitment_chip.hash([witness, 0])
   expose_public(commitment_digest, position=1)
   ```

3. **Second Hash - Nullifier:**
   ```
   poseidon_chip = Pow5Chip (Poseidon hasher)
   nullifier_digest = poseidon_chip.hash([witness, nonce])
   expose_public(nullifier_digest, position=0)
   ```

4. **Public Output:**
   - `public[0]` = nullifier
   - `public[1]` = commitment

### Why Two Separate Poseidon Instances?

The circuit uses two different Poseidon configurations:
- `commitment_hash_config` - For hashing witness with 0
- `hash_config` - For hashing witness with nonce

This is a circuit design pattern to separate different uses of the hash function.

---

## 3️⃣ The Halo2 Proving System

### What is Halo2?

Halo2 is a zero-knowledge proof system that allows Bob to prove:
- "I know values X and Y such that some computation on X and Y produces result Z"
- Without revealing X or Y
- The verifier only sees Z and the proof

### Components Used

1. **Constraint System:**
   - Defines what needs to be proven
   - Built using gates and custom constraints
   - In this case: two Poseidon hash computations

2. **Polynomial Commitment Scheme: KZG**
   - Commits to polynomials representing the computation
   - Uses elliptic curve BN256
   - Allows verification without seeing the polynomial

3. **Multi-Opening Scheme: SHPLONK**
   - Efficiently proves multiple polynomial evaluations
   - Reduces proof size
   - Part of the Halo2 protocol

### Proof Generation Flow

```rust
// 1. Setup phase (trusted setup)
params = KZG_Setup(K=6)  // Circuit size parameter

// 2. Key generation
vk = keygen_vk(params, circuit_structure)
pk = keygen_pk(params, vk, circuit_structure)

// 3. Proof generation
proof = create_proof(
    params,
    pk,
    circuit_with_witness,  // Including secret and nonce
    public_inputs           // Including nullifier and commitment
)

// 4. Verification
verify_proof(
    params,
    vk,
    proof,
    public_inputs
)
```

---

## 4️⃣ The Poseidon Hash Function

### What is Poseidon?

Poseidon is a cryptographic hash function designed specifically for use in zero-knowledge circuits:
- **Algebraically efficient:** Uses fewer constraints than SHA-256
- **Secure:** Designed to resist algebraic attacks
- **Optimized for ZK:** Arithmetic operations over finite fields

### Configuration Used

```rust
P128Pow5T3<Fr>
```

Meaning:
- **P128:** Security level (128-bit)
- **Pow5:** Uses x^5 as the S-box (non-linear operation)
- **T3:** State size t=3 (width of the permutation)

**Parameters:**
- `WIDTH = 3`: Total state size
- `RATE = 2`: How many field elements can be absorbed per iteration
- `CAPACITY = 1`: Security parameter (WIDTH - RATE)

### How Poseidon Hashing Works

```rust
// Hash two field elements [a, b]
hash = Poseidon([a, b])

// Internal process:
1. Initialize state: [0, 0, 0]
2. Absorb inputs: state[0] = a, state[1] = b
3. Apply permutation (multiple rounds):
   - Add round constants
   - Apply S-box (x^5)
   - Mix (matrix multiplication)
4. Squeeze output: return state[0]
```

### Why Can't We Reverse It?

Poseidon is:
- **One-way:** Given `hash`, finding `[a, b]` is computationally infeasible
- **Collision-resistant:** Hard to find different `[a, b]` and `[c, d]` with same hash
- **Pseudorandom:** Output looks random, no patterns

---

## 5️⃣ The 64 Proofs

### How They Were Generated

```rust
serialize() {
    for i in 0..64 {
        secret = <SAME_SECRET_EVERY_TIME>
        nonce = Fr::random(OsRng)  // Different each time!
        
        commitment = Poseidon([secret, 0])
        nullifier = Poseidon([secret, nonce])
        
        proof = create_proof(secret, nonce)
        
        save("proof_{i}.bin", proof)
        save("nullifier_{i}.bin", nullifier)
        save("commitment_{i}.bin", commitment)
    }
}
```

### What This Means

**All 64 proofs:**
- ✅ Use the SAME secret
- ✅ Use DIFFERENT random nonces
- ✅ Have the SAME commitment (Poseidon(secret, 0))
- ✅ Have DIFFERENT nullifiers (Poseidon(secret, nonce_i))
- ✅ Are all valid proofs

**The Data:**
- 64 valid ZK proofs
- 64 unique nullifiers (all different)
- 64 identical commitments (all the same)
- All verifiable and correct

---

## 6️⃣ Why Bob's Proofs Are Failing

### The Mystery

Bob's proofs are **mathematically valid** but get **rejected by SuperCoolAirdrop™**.

Why? The puzzle story suggests SuperCoolAirdrop™ might be malicious!

### The Real Question

The puzzle isn't about why proofs are rejected. It's about:

**Can an attacker (or malicious service) recover Bob's secret from the public data?**

Public data available:
- ✅ 64 proofs (binary blobs)
- ✅ 64 nullifiers (field elements)
- ✅ 64 commitments (all identical)
- ❌ Secret (private)
- ❌ Nonces (private)

### The Security Assumption

**IF the system is secure:**
- Seeing 64 pairs of `(nullifier_i, commitment)` should not reveal the secret
- ZK proofs should not leak information about witnesses
- Even with unlimited computing power, extracting secret should be infeasible

**IF there's a vulnerability:**
- Secret could be guessable (small value, timestamp, etc.)
- Proofs might leak information
- Hash function might have weakness with multiple samples
- Implementation bug could expose data

---

## 7️⃣ The Attack Surface

### What You Can Analyze

1. **Commitments (all identical):**
   ```
   0x1f2b5aac96a2527339892fca32dd80b2eb4132b5f90acf27861cba5ddd6ddacf
   ```
   - This is `Poseidon(secret, 0)`
   - If you guess `secret`, you can verify by computing `Poseidon(guess, 0)`

2. **Nullifiers (all unique):**
   ```
   nullifier_0: 0x1e7b658c47c8006f615fa60156ce8556907001993eda2cb0b387b59d38634722
   nullifier_1: 0x09ad7f6cf1d37f9a53a39e5cf6e5c506631096d8d900965a47992cbedc1d6ba3
   ...
   ```
   - These are `Poseidon(secret, nonce_i)` for random `nonce_i`
   - 64 different hash outputs using the same secret

3. **Proofs (binary data):**
   - ZK proof blobs
   - Should not leak witness information (if secure)
   - Could analyze for patterns/size/structure

### Attack Strategies

**Strategy 1: Brute Force the Secret**
```rust
for guess in possible_secrets {
    if Poseidon([guess, 0]) == commitment {
        return guess;  // Found it!
    }
}
```

**Strategy 2: Algebraic Analysis**
- Try to extract secret from multiple nullifier equations
- Look for polynomial relationships
- Exploit hash function structure (if vulnerable)

**Strategy 3: Side-Channel Analysis**
- Analyze proof bytes
- Look for timing patterns (not applicable here)
- Search for implementation bugs

---

## 8️⃣ The Solution Approach

### What the Skeleton Code Does

```rust
let secret = Fr::from(0u64);  // Guess: secret = 0
let test_commitment = Poseidon([secret, Fr::from(0)]);

for i in 0..64 {
    let (_, _, commitment) = from_serialized(i);
    assert_eq!(test_commitment, commitment);  // Check if guess is right
}
```

**Result:** FAILS because secret ≠ 0

### What You Need to Do

**Find the actual secret value!**

Modify the code to:
1. Try different secret values
2. For each guess, compute `Poseidon(guess, 0)`
3. Compare with actual commitment
4. When they match → you found Bob's secret!

---

## 9️⃣ Files Explained

### Source Code

- **`src/main.rs`:** Main puzzle code
  - Circuit definition (`MyCircuit`)
  - Proof generation (`create_proof`)
  - Proof verification (`verify_proof`)
  - Data loading (`deserialize`)
  - Attack code (to be implemented)

### Libraries

- **`halo2/`:** Halo2 proving system
  - Core ZK proof infrastructure
  - Constraint system
  - Polynomial commitments

- **`poseidon-circuit/`:** Poseidon hash
  - `poseidon-base`: Pure Rust implementation
  - `poseidon-circuit`: Halo2 circuit gadget

### Data Files

- **`commitments/commitment_{0-63}.bin`:** 
  - All contain same value: `Poseidon(secret, 0)`
  - 32 bytes each (Fr field element)

- **`nullifiers/nullifier_{0-63}.bin`:**
  - Each unique: `Poseidon(secret, nonce_i)`
  - 32 bytes each (Fr field element)

- **`proofs/proof_{0-63}.bin`:**
  - ZK proofs (SHPLONK format)
  - Variable size (~1-2 KB each)

---

## 🎯 Summary

**The System:**
- Airdrop claiming using ZK proofs
- Proves knowledge of secret without revealing it
- Uses Poseidon hash for commitments and nullifiers

**The Data:**
- 64 valid proofs from Bob
- All use same secret, different nonces
- Commitments identical, nullifiers unique

**The Challenge:**
- Find Bob's secret from public data
- Likely involves guessing/brute-forcing
- Puzzle name "Zeitgeist" hints at time-related secret

**The Goal:**
- Compute `Poseidon(guess, 0)` for various guesses
- Find guess where output matches `0x1f2b5aac96a2527339892fca32dd80b2eb4132b5f90acf27861cba5ddd6ddacf`
- That guess is Bob's secret!

---

**Ready to solve it? The secret is waiting to be discovered! 🔍**

