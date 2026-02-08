---
layout: post
title: Threshold ECDSA Protocols - Evolution, Security Models, and Operational Implementation
date: 2025-08-10
description: Analysis of various MPC protocols
tags: EA
categories: EA
related_posts: false
thumbnail: assets/img/mpc/cover.png
---

## 1. Introduction to Threshold Cryptography in Digital Asset Custody

The rapid institutionalization of digital assets has necessitated a fundamental shift in cryptographic key management. Traditional single-signature schemes, where a private key resides on a single device or in a single memory space, present an unacceptable single point of failure for high-value custody operations. While multi-signature (multisig) schemes offer a solution by requiring $M$-of-$N$ on-chain signatures, they suffer from privacy leakage (access structures are visible on the public ledger), higher transaction fees (larger witness data), and a lack of protocol agility (different blockchains implement multisig differently, or not at all).

This operational landscape drove the development of Threshold Signature Schemes (TSS) based on Multi-Party Computation (MPC). In a threshold scheme, the private key $x$ is never reconstructed in a single location. Instead, it is generated and used in a distributed manner by $n$ parties, such that any subset of $t+1$ parties can validly sign a transaction, while any subset of size $t$ or smaller learns nothing about the key.

While threshold schemes for Schnorr signatures (and by extension, EdDSA) are relatively straightforward due to the linearity of the signing equation ($s = k + ex$), the Elliptic Curve Digital Signature Algorithm (ECDSA)—the standard for Bitcoin and Ethereum—presents a formidable mathematical challenge. ECDSA signing requires the computation of the modular inverse of a secret ephemeral nonce, $k^{-1}$, and the multiplication of two secret values, $k \cdot x$. In a distributed setting, performing these non-linear operations on secret shares without revealing the underlying values requires complex cryptographic machinery.

This artcile provides a comprehensive technical analysis spanning the complete evolution of threshold signature schemes: from Shamir's Secret Sharing (the foundational academic baseline), through the modern MPC era with GG18 (Gennaro & Goldfeder 2018), GG20 (Gennaro & Goldfeder 2020), CGGMP21 (Canetti et al. 2021), and MPC-CMP (Fireblocks/CMP), to emerging 2025-2026 protocols including DHSMPC/nQSMax and FROST. We dissect their cryptographic primitives, round-by-round signing mechanics, security models (from static to adaptive adversaries), the critical vulnerabilities—such as the Paillier modulus attacks—that drove their evolution, and the emerging quantum-safe directions. Furthermore, we analyze the integration of these software protocols with hardware-based Trusted Execution Environments (TEEs) and their application in broader privacy-preserving domains like Federated Learning and AI/ML systems.

**Protocol Evolution Timeline:**

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/mpc/1.png" class="img-fluid rounded z-depth-1" zoomable=true %}
    </div>
</div>

## 2. Cryptographic Primitives and the MtA Paradigm

To understand the architectural distinctions between GG18, GG20, and CGGMP21, one must first master the underlying cryptographic primitives that enable the distributed computation of ECDSA signatures. The core difficulty lies in the Multiplicative-to-Additive (MtA) conversion, a sub-protocol used in all four schemes to transform multiplicative shares of a secret into additive shares without revealing the secret itself.

### 2.0 Background: ECDSA in the Standard (Non-Threshold) Case

Before discussing threshold ECDSA, it's essential to understand how ECDSA works in its standard form, used by Bitcoin and Ethereum for all transaction signatures.

**What is ECDSA?**

ECDSA (Elliptic Curve Digital Signature Algorithm) is the cryptographic standard for asymmetric digital signatures on elliptic curves. Unlike RSA (which uses large prime factorization), ECDSA relies on the mathematical difficulty of the **Elliptic Curve Discrete Logarithm Problem (ECDLP)**: given a point $Q$ on the curve and a generator point $G$, it's computationally infeasible to find the scalar $x$ such that $Q = x \cdot G$.

**Key Components:**
- **Private Key:** A scalar $x$ (256-bit number for Bitcoin/Ethereum, e.g., $x \in [1, n-1]$ where $n$ is the curve order)
- **Public Key:** The curve point $Q = x \cdot G$ (computed from the private key; revealed to the world)
- **Generator Point $G$:** A fixed point on the elliptic curve (standardized for each curve like secp256k1)
- **Curve Order $n$:** The number of points on the curve (approximately $2^{256}$ for secp256k1)

**Standard ECDSA Signing (Single Party):**

When a single user wants to sign a message $m$:

1. **Hash the Message:** $H(m)$ = cryptographic hash of the message (using SHA-256)
2. **Generate Ephemeral Nonce:** Select a random secret nonce $k \in [1, n-1]$ (must be different for each signature)
3. **Compute Curve Point:** $R = k \cdot G$ (scalar multiplication on the elliptic curve)
4. **Extract x-coordinate:** $r = R_x \pmod n$ (use the x-coordinate of point $R$, reduced modulo $n$)
5. **Compute Signature Component:** $s = k^{-1}(H(m) + r \cdot x) \pmod n$
   - $k^{-1}$ is the modular multiplicative inverse (the value that when multiplied by $k$ gives 1 modulo $n$)
   - This equation combines the message hash, the ephemeral nonce, and the private key

6. **Return Signature:** $(r, s)$ is the final signature (64 bytes total: 32 for $r$, 32 for $s$)

**Signature Verification (Anyone with Public Key):**

Anyone can verify a signature without knowing the private key:

$$Q \stackrel{?}{=} s^{-1}(H(m) \cdot G + r \cdot Q)$$

If this equation holds, the signature is valid. This works because of the algebraic properties of elliptic curves and modular arithmetic.

**Why ECDSA is Secure:**

The security rests on the difficulty of the discrete logarithm problem. An attacker cannot forge a signature for a different message without knowing $x$ (the private key), because they would need to solve $x = ?$ given only $Q$ and $G$, which is computationally infeasible.

### 2.1 The ECDSA Threshold Challenge

Now we move to the **threshold setting**, where the private key $x$ is not held by a single party but is split among $n$ parties. This creates a fundamental mathematical challenge.

**The Threshold Equation:**

In threshold ECDSA, the signature equation must be computed on shares rather than on the key itself:

- **Curve Point:** $R = k \cdot G = (r_x, r_y)$ where $k = \sum_{i=1}^{n} k_i$ (sum of all nonce shares)
- **Signature Component 1:** $r = r_x \pmod n$ (extract x-coordinate as in standard ECDSA)
- **Signature Component 2:** $s = k^{-1} (H(m) + r \cdot x) \pmod n$ where $x = \sum_{i=1}^{n} x_i$ (sum of all private key shares)

**Why This is Hard:**

In a threshold setting, the private key $x$ is shared additively among participants: $x = \sum_{i=1}^{n} x_i$ (each party $i$ holds only their share $x_i$, not the full key).

Similarly, the ephemeral nonce $k$ is generated via distributed shares: $k = \sum_{i=1}^{n} k_i$ (each party generates part of the nonce).

The signing equation requires two **non-linear operations** on these shares that cannot be done naively:

1. **Inversion Problem:** Computing $k^{-1}$ from shares of $k$
   - If we try $k^{-1} = \sum_{i=1}^{n} (k_i)^{-1}$, this is **WRONG**
   - Mathematically: $(\sum k_i)^{-1} \neq \sum (k_i)^{-1}$ (modular inverse is non-linear)
   - Example: If $k_1 = 2$ and $k_2 = 3$, then $(2+3)^{-1} \neq 2^{-1} + 3^{-1}$ in modular arithmetic

2. **Multiplication Problem:** Computing $k \cdot x$ from their respective shares
   - If we try $k \cdot x = \sum_{i=1}^{n} (k_i \cdot x_i)$, this is also **WRONG**
   - Mathematically: $(\sum k_i)(\sum x_i) \neq \sum (k_i \cdot x_i)$ (product expands differently)
   - Example: $(k_1 + k_2)(x_1 + x_2) = k_1 x_1 + k_1 x_2 + k_2 x_1 + k_2 x_2$ (cross terms!)

**The Solution: Homomorphic Encryption**

To solve this, the protocols utilize **Additively Homomorphic Encryption (AHE)**—specifically Paillier encryption—to perform operations on encrypted data in a way that parties can compute the product $k \cdot x$ on their shares without revealing the actual values to each other.

### 2.2 Paillier Homomorphic Encryption

The Paillier cryptosystem, invented by Pascal Paillier in 1999, is a probabilistic asymmetric algorithm for public-key cryptography. Unlike standard encryption schemes (which typically only allow decryption and not computation on encrypted data), Paillier allows **computation directly on encrypted data without decryption**—a property crucial for MPC.

**Intuition: Why Homomorphic Encryption Matters**

Normally, encryption is like locking information in a box:
- You can lock it (encryption)
- Only the key holder can unlock it (decryption)
- But you **cannot do arithmetic inside the locked box**

Homomorphic encryption is special: you **can perform arithmetic on locked (encrypted) values** without ever unlocking them. The result is an encrypted answer that, when decrypted, gives the correct arithmetic result.

**Example:** If Alice encrypts 5 and Bob encrypts 3:
- Standard encryption: Only the owner can decrypt; Bob can't know that Alice has 5
- Paillier homomorphic: Bob can encrypt something with Alice's public key, add the encrypted 5 to his encrypted 3, and send back an encrypted 8. When Alice decrypts, she gets 8 (the correct sum) without Bob ever seeing 5 or learning the private key.

**Mathematical Foundation:**

Paillier uses two large prime numbers to create the cryptographic structure:

- **Key Generation:** 
  - Select two large primes $p, q$ (typically 1024-2048 bits each)
  - Compute $N = pq$ (the public key; this becomes publicly known)
  - Compute $\lambda(N) = \text{lcm}(p-1, q-1)$ (the private key; known only to the key holder)
  - The security rests on the difficulty of factoring $N$ (knowing $N$ but not $p, q$ is hard)

- **Encryption:** To encrypt a message $m$ (any number less than $N$):
  - Select random $r \in \mathbb{Z}_N^*$ (a random number coprime to $N$)
  - Compute ciphertext: $c = g^m \cdot r^N \pmod{N^2}$
  - The randomness $r$ is essential: encrypting the same message twice gives different ciphertexts (probabilistic encryption)
  - This makes the encryption semantically secure (an attacker cannot tell if two ciphertexts encrypt the same message)

- **Decryption:** Using the private key $\lambda(N)$:
  - Compute $m = D(c)$ using a specific formula involving $\lambda$ and the ciphertext
  - Only the holder of $\lambda$ can decrypt; without it, the computation is infeasible
  - The decryption is deterministic: the same ciphertext always decrypts to the same plaintext

**Why Paillier is Additive (and Multiplicative with Scalars):**

The magic of Paillier comes from its algebraic structure. Two encrypted values can be added in the ciphertext domain:

- **Addition of Ciphertexts:** If $c_1 = E(m_1)$ and $c_2 = E(m_2)$, then:
  $$E(m_1) \cdot E(m_2) \pmod{N^2} = E(m_1 + m_2)$$
  Multiply the two ciphertexts, decrypt, and you get the sum of the two messages!

- **Scalar Multiplication:** If $c = E(m)$ and $k$ is a known scalar (not encrypted):
  $$E(m)^k \pmod{N^2} = E(k \cdot m)$$
  Raise the ciphertext to the power of $k$, decrypt, and you get the product!

**Example (Concrete Numbers):**
Let's say Alice encrypts 7 and Bob encrypts 3 with Paillier:
- $E_A(7) = c_1$ (some large encrypted value)
- $E_A(3) = c_2$ (some other encrypted value)
- Compute: $c_1 \cdot c_2 \pmod{N^2} = E_A(7 + 3) = E_A(10)$
- When Alice decrypts: she gets 10 (the sum!)
- Bob never saw 7 or the private key; he just did multiplication in the ciphertext domain

**Security Assumption:**

Paillier's security rests on the **Decisional Composite Residuosity Assumption (DCRA)**:

Given $N$ and a random number $z \in \mathbb{Z}_{N^2}^*$, it is computationally infeasible to determine whether $z$ is a Paillier encryption of 0 or some random non-zero value. This is an **assumption** (unproven conjecture) like the discrete logarithm problem, but believed to be hard based on decades of research.

**Practical Characteristics:**

- **Ciphertext Expansion:** A $k$-bit plaintext expands to $2k$ bits when encrypted (e.g., a 256-bit secret becomes 512-bit encrypted). This overhead is acceptable for MPC because we're not dealing with massive data.
- **Computational Cost:** 
  - Encryption is $O(k^3)$ (expensive due to modular exponentiation with large numbers)
  - Homomorphic operations are also $O(k^3)$ (similarly expensive)
  - Decryption is $O(k^3)$
  - These are expensive compared to symmetric encryption, but manageable for threshold signature signing (which happens once per transaction)
- **Parallelizability:** The modular exponentiations can be parallelized, so modern systems can handle ~10-20ms per Paillier operation

**Why Paillier for Threshold ECDSA?**

The scalar multiplication property ($E(m)^k = E(k \cdot m)$) is exactly what we need to solve the multiplication problem in threshold ECDSA:
- Alice has secret $a$ and can encrypt it with Paillier
- Bob has secret $b$ (not encrypted)
- Bob can compute $E(a)^b = E(a \cdot b)$ without ever learning $a$
- Alice can decrypt and get $a \cdot b$
- This is the core of the **Multiplicative-to-Additive (MtA) protocol** that appears in all threshold ECDSA schemes

### 2.3 The Multiplicative-to-Additive (MtA) Protocol

The MtA protocol is the "engine" of threshold ECDSA. It allows two parties, Alice (holding secret $a$) and Bob (holding secret $b$), to compute additive shares $\alpha$ and $\beta$ such that $\alpha + \beta = a \cdot b$.

**Protocol Flow:**

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/mpc/2.png" class="img-fluid rounded z-depth-1" zoomable=true %}
    </div>
</div>

**The Protocol Steps in Detail:**

1. **Encryption:** Alice generates a Paillier key pair $(N, p, q)$ where the public key is $N$ and private key is the factorization. She encrypts her secret $a$ and sends $c_A = E_A(a)$ to Bob. Importantly, Alice does **not** reveal the Paillier modulus's factorization or any information that would allow Bob to decrypt.

2. **Homomorphic Computation:** Bob receives $c_A$ and performs homomorphic operations:
   - He selects a random blinding value $\beta \in [0, 2^{\ell}]$ where $\ell$ is a security parameter (typically 256+ bits)
   - Using Paillier's scalar multiplication property: $c_B = c_A^b \pmod{N^2}$ computes the encryption of $a \cdot b$
   - He then adds encryption of his blinding factor: $c_B' = c_B \otimes E_A(\beta)$, resulting in $E_A(ab + \beta)$
   - The blinding factor is critical: without it, Alice would learn $ab$ upon decryption
   - Bob sends $c_B'$ back to Alice

3. **Decryption:** Alice decrypts using her private key:
   - She computes $\alpha' = D_A(c_B') = ab + \beta \pmod{n}$ (where $n$ is the ECDSA curve order)
   - She adjusts: $\alpha = -\alpha' \pmod{n}$ (the sign depends on the signing equation variant)
   - Alice now holds her share $\alpha$

4. **Result:** Alice has $\alpha = -(ab + \beta)$, Bob has $\beta$
   - Addition: $\alpha + \beta = -(ab + \beta) + \beta = -ab \pmod{n}$
   - The product is shared in the form needed for the signing equation: $ab$ is decomposed into two shares that only reveal the product when summed

5. **Security via Zero-Knowledge Proofs:** This exchange must be protected with rigorous ZKPs:
   - **Range Proof (Alice):** Proves that her encrypted value $a$ is within a valid range $[0, q^3]$ (where $q$ is the curve order). This prevents overflow attacks where an attacker sends a value $> q$ that wraps around modulo $N$
   - **Correctness Proof (Bob):** Proves that Bob correctly computed the homomorphic multiplication and that his blinding factor $\beta$ is within valid bounds
   - **Modulus Proof (Alice):** At key generation time, Alice proves that her Paillier modulus $N$ is a product of exactly two safe primes (no small factors that weaken the encryption)

The failure to rigorously enforce these proofs was the source of the devastating Alpha-Rays and 6ix1een attacks, discussed later in Section 9.[1, 3]

**Computational Complexity:**
- Alice's cost: 1 Paillier encryption ($O(k^3)$ where $k = 2048$), 1 decryption ($O(k^3)$), ZK proof computation (~1-2ms)
- Bob's cost: 1 Paillier homomorphic operation ($O(k^3)$), 1 Paillier addition ($O(k^2)$), ZK proof (~1-2ms)
- Total per MtA invocation: **~10-20ms** per party (dominant cost in GG18/GG20 signing)

## 3. Shamir's Secret Sharing: The Academic Foundation

Shamir's Secret Sharing (SSS), invented by Adi Shamir in 1979, represents the foundational building block of all modern threshold cryptography. While elegant in its mathematical simplicity, it reveals the critical limitations that drove the development of distributed key generation and practical MPC protocols.

### 3.1 Protocol Architecture

Shamir's scheme uses polynomial interpolation to distribute a secret $s$ among $n$ parties such that any $t+1$ parties can reconstruct it, but any $t$ parties learn nothing.

**Visual Overview of Shamir's Secret Sharing:**

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/mpc/12.png" class="img-fluid rounded z-depth-1" zoomable=true %}
    </div>
</div>

**Protocol Steps:**

1. **Polynomial Generation:** A dealer selects a random polynomial of degree $t-1$:
   $$f(x) = s + a_1 x + a_2 x^2 + \cdots + a_{t-1} x^{t-1} \pmod p$$
   The constant term is the secret: $f(0) = s$. The coefficients $a_1, ..., a_{t-1}$ are random and secret (never revealed).

2. **Share Distribution:** The dealer evaluates the polynomial at $n$ distinct non-zero points and distributes shares:
   - Party $i$ receives share $s_i = f(i) \pmod p$
   - Each party stores a single value; **no one except the dealer ever sees the polynomial or understands how shares relate**
   - The distribution must occur over a secure channel to prevent eavesdropping

3. **Reconstruction (Lagrange Interpolation):** Any $t+1$ parties can recover the secret by computing:
   $$s = \sum_{i=0}^{t} s_i \cdot L_i(0) \pmod p$$
   where $L_i(x)$ is the Lagrange basis polynomial:
   $$L_i(x) = \prod_{j \neq i} \frac{x - x_j}{x_i - x_j}$$
   The original secret is recovered without reconstructing intermediate values. This reconstruction is **deterministic**—there is only one value of $s$ that satisfies the polynomial equation with $t+1$ shares.

**Information-Theoretic Security:**
- **Perfect Secrecy:** Any subset of $\leq t$ shares reveals **zero information** about the secret. This is information-theoretic (not computational)—no amount of computational power can recover the secret from $t$ shares. This holds even against quantum computers.
- **Why:** With $t$ shares and a degree $t-1$ polynomial, there are infinitely many polynomials consistent with those $t$ shares. Each consistent polynomial could correspond to any possible secret value $s'$, making the secret space equally likely.
- Trade-off: $t$ larger means more security (more parties needed to compromise), but also more communication overhead for reconstruction

### 3.2 Why Shamir Fails at Scale: Five Critical Limitations

While mathematically sound, Shamir's scheme cannot be directly applied to real-world threshold ECDSA without substantial additional machinery:

1. **Trusted Dealer Problem:** A single dealer generates all shares. If the dealer is compromised, malicious, or performs "biased sharing," they can extract any $t$ shares and reconstruct the secret later. Modern protocols solve this via Distributed Key Generation (DKG), eliminating the dealer entirely.

2. **Expensive Reconstruction:** To sign, all parties must reconstruct the private key in memory—a massive security violation. In practice, reconstructing a 256-bit key via Lagrange interpolation in $\mathbb{Z}_p$ (where $p \approx 2^{256}$) requires:
   - Modular inversions: $t+1$ inversions, each ~500-800μs
   - Modular multiplications: $(t+1)^2$ multiplications
   - Total latency: **500-800ms per signature** for $t=3$
   - Maximum throughput: **2 signatures/second** (unacceptable for trading or payment processing)

3. **No Proof of Correctness:** Shamir's scheme provides no mechanism to verify that shares are correctly formed. A dishonest dealer can distribute corrupted shares, and parties won't detect this until reconstruction fails. The protocol offers no accountability.

4. **Anonymous Abort & DoS Vulnerability:** If reconstruction fails (due to a bad share), the protocol simply outputs an error. There is no way to identify which party provided the bad share. A single malicious party can perform indefinite Denial-of-Service attacks by contributing corrupted shares.

5. **Key Reconstruction Requirement:** The secret must be fully reconstructed to use it for signing. This violates the core principle of threshold schemes: **the secret should never exist in unencrypted form on any single device**. Even during signing, parties must keep their shares secret and use distributed computation.

### 3.3 Empirical Performance Limitations

- **Signing Latency:** 500-800ms per signature (in practice, dominated by reconstruction time)
- **Throughput:** ~2-3 signatures/second maximum
- **Scalability:** Reconstruction complexity grows quadratically with $n$ (number of parties)
- **Security Model:** Requires a trusted dealer; no protection against adaptive adversaries
- **Key Reuse:** The same key shares remain valid indefinitely; no native proactive refresh

### 3.4 Transition to Modern MPC: Why GG18 Was Necessary

The limitations of Shamir's scheme motivated the development of GG18, which introduced:
- **Trustless DKG** via Verifiable Secret Sharing (VSS), eliminating the trusted dealer
- **Distributed signing** that never reconstructs the secret key
- **Identifiable abort** mechanisms (added in GG20) to prevent anonymous DoS
- **Constant-round protocols** reducing latency from hundreds of milliseconds to seconds
- **Formal security proofs** under realistic adversarial models

The insight was that threshold cryptography could be made practical only by moving away from polynomial reconstruction and toward interactive MPC, where parties collaborate without ever exposing the key.

## 4. GG18: The Foundational Protocol

GG18 (Gennaro & Goldfeder, 2018) represented a watershed moment in threshold cryptography. It was the first efficient, constant-round protocol for threshold ECDSA that did not require a trusted dealer during key generation, making it suitable for permissionless blockchain applications.

### 4.1 Protocol Architecture and Security Model

GG18 operates within a rigorous cryptographic framework designed to enable trustless threshold signing:

**Adversarial Model:**
- **Static Malicious Corruption:** The adversary must designate which $t$ parties to corrupt before key generation begins. This is weaker than "adaptive" security (where an attacker can corrupt parties mid-protocol), but stronger than semi-honest models.
- **Computational Security:** The protocol is secure against polynomial-time adversaries (assuming no breakthroughs in discrete logarithm computation or Paillier factorization).
- **Threshold Assumption:** Any subset of $t$ or fewer parties learns nothing about the private key. Any subset of $t+1$ parties can validly sign.

**Network Model:**
- **Asynchronous P2P:** Point-to-point secure channels between all pairs of parties (assuming a PKI or out-of-band authentication).
- **Broadcast:** Parties can broadcast messages (via a bulletin board or gossip protocol). Broadcast security is not assumed to be instant.
- **Synchrony Assumption:** For DKG, rounds are synchronous (all parties proceed to the next round simultaneously). For signing, rounds are asynchronous (parties can proceed at different times).

**Cryptographic Assumptions:**
- **DCRA (Decisional Composite Residuosity Assumption):** Paillier homomorphic encryption security
- **DDH (Decisional Diffie-Hellman):** Elliptic curve operations
- **ECDSA Unforgeability:** Standard elliptic curve digital signature security

**Performance Target:**
- **Key Generation:** O(n²) communication, ~10-30 seconds for n=5 parties
- **Signing:** 9 rounds total, ~8-10 seconds latency for n=5 parties, t=2

### 4.2 Distributed Key Generation (DKG)

The GG18 KeyGen phase replaces the need for a trusted dealer by combining Feldman's Verifiable Secret Sharing (VSS) with commitment schemes.

**Feldman's Verifiable Secret Sharing (VSS) Overview:**

In Feldman VSS, a party $P_i$ sharing a secret $u_i$ creates a polynomial:
$$f_i(x) = u_i + a_{i,1} \cdot x + a_{i,2} \cdot x^2 + ... + a_{i,t} \cdot x^t \pmod p$$

The key innovation is publishing "verification values" (commitments to the coefficients):
$$V_{i,j} = a_{i,j} \cdot G \quad \text{(elliptic curve points)}$$

Any party can verify that the share they received, $s_{i,j} = f_i(j)$, is correct by checking:
$$s_{i,j} \cdot G = \sum_{k=0}^{t} V_{i,k} \cdot j^k$$

This verification is done in the exponent (on the elliptic curve), revealing no information about the secret $u_i$.

**Round-by-Round DKG Breakdown:**

1. **Commitment (Round 1):** Each player $P_i$ performs the following:
   - Generates Paillier key pair: $N_i = p_i \cdot q_i$ (2048-bit modulus typical)
   - Generates commitments to their polynomial coefficients for later opening
   - Broadcasts a cryptographic hash commitment: $H(N_i \parallel C_i)$ where $C_i$ are Feldman commitments
   - Stores the preimage for later decommitment

2. **Decommitment (Round 2):** Parties reveal:
   - Full Paillier public key $N_i$
   - Feldman VSS commitments $V_{i,0}, V_{i,1}, ..., V_{i,t}$ (points on elliptic curve)
   - **ZK Proof of Modulus:** A zero-knowledge proof that $N_i$ is a product of two "safe primes" $p_i = 2p'_i + 1$ and $q_i = 2q'_i + 1$. This proof ensures:
     - $N_i$ has no small factors
     - The modulus is "biprime" (exactly two prime factors)
     - The proof uses roughly 10-20 rounds of challenge-response to achieve ~2^{-80} soundness error
     - Computational cost: ~100-500ms per party for the prover

3. **VSS Share Distribution (Round 3):** Each $P_i$ distributes their polynomial evaluations:
   - For each other party $P_j$, compute and send the encrypted share: $E_{i,j} = E_{N_i}(f_i(j))$
   - Send the Feldman verification values for all parties to verify: $V_{i,0}, V_{i,1}, ..., V_{i,t}$
   - **Verification Check:** Each $P_j$ verifies they received the correct share by checking:
     $$E_{i,j} \stackrel{?}{\equiv} \prod_{k=0}^{t} V_{i,k}^{j^k} \pmod{N_i^2}$$
   - If verification fails, party $P_j$ broadcasts a "complaint" that $P_i$ is faulty

4. **Handling Complaints (Round 4):** If a complaint is raised:
   - The accused party $P_i$ publishes the share in the clear (or in an encrypted form with a proof)
   - Parties verify whether the share was indeed incorrect
   - If $P_i$ cannot resolve the complaint, they are disqualified and their shares are ignored

5. **Local Key Derivation (Round 5):** Each party $P_i$ computes their final private key share:
   $$x_i = \sum_{j=1}^{n} s_{j,i}$$
   where $s_{j,i} = f_j(i)$ is the share received from party $P_j$.

   The global public key is:
   $$Q = \sum_{j=1}^{n} V_{j,0}$$
   (the sum of all constant coefficients on the elliptic curve)

**Complexity Analysis of GG18 DKG:**

| Component | Cost per Party | Total (n=5) |
|-----------|----------------|------------|
| Paillier Key Generation | ~100ms | ~500ms |
| Feldman Commitment generation | ~10ms | ~50ms |
| ZK Proof of Modulus (10-20 rounds) | ~200-500ms | ~1-2.5s |
| Share distribution & verification | ~50ms | ~250ms |
| Complaint handling (if needed) | ~10-50ms | ~50-250ms |
| **Total DKG Time** | ~400-700ms per party | ~2-4 seconds |

**Verification Values: Why They Matter:**

The Feldman verification values $V_{i,k} = a_{i,k} \cdot G$ serve a dual purpose:
1. **Public Verifiability:** Any party can verify that their received share $s_{i,j}$ satisfies the polynomial relation without knowing the secret $u_i$
2. **Theft Detection:** If a share is tampered with during transmission, the verification check will fail, and the tampering is detectable
3. **Non-Interactive Audit:** Even months later, an auditor can verify that the DKG was performed correctly by inspecting the published verification values

The total data published for DKG: ~n × (t+1) elliptic curve points + n × 2048-bit Paillier moduli ≈ **20-50KB for n=10, t=3**

### 4.3 The Signing Protocol (9 Rounds)

The signing protocol transforms the additive shares of $x$ into a valid signature $(r, s)$ without reconstructing $x$.

**9-Round GG18 Signing Protocol Visualization:**

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/mpc/3.png" class="img-fluid rounded z-depth-1" zoomable=true %}
    </div>
</div>

**Phase Breakdown:**

The protocol flows left-to-right across three logical phases:
- **Phase 1 (Rounds 1-2):** Ephemeral key generation - parties commit and exchange encrypted nonce shares
- **Phase 2 (Rounds 3-6):** Inversion & point calculation - distributed computation of R and r values
- **Phase 3 (Rounds 7-9):** Signature assembly - compute and verify final signature

**Communication Pattern Analysis:**

- **Total Messages:** ~40-50 (each Round 2 MtA requires $\binom{n}{2} \times 2$ messages for $n$ parties)
- **Bandwidth per Signing:** For $n=5$ parties, ~500KB of ZK proofs and commitments
- **Parallelization:** Rounds 1-3 can be partially parallelized; Rounds 4-6 require strict sequencing due to inversion dependencies
- **Bottleneck:** Round 2 MtA exchanges (all-to-all communication O(n²) messages)

**Phase 1: Ephemeral Key Generation (Nonce $k$)**

- **Round 1:** Parties commit to ephemeral secrets $k_i$ (for the nonce) and $\gamma_i$ (a masking scalar).

- **Round 2:** Parties engage in the MtA Protocol. Every pair $(P_i, P_j)$ runs MtA twice:
  - To compute additive shares of $k \cdot \gamma$ (where $k = \sum k_i, \gamma = \sum \gamma_i$).
  - To compute additive shares of $k \cdot w$ (where $w$ is the private key share scalar).
  - This all-to-all communication is bandwidth-intensive.[1]

**Phase 2: Inversion and Point Calculation**

- **Rounds 3-6:** The parties essentially perform a distributed inversion of $k$. They reveal the value $\Gamma = \sum \gamma_i$ and use the multiplicative shares derived in Round 2 to compute the curve point $R = (\sum k_i)^{-1} \cdot G$. By revealing $\Gamma$ but keeping shares of $k \cdot \gamma$ secret until the conversion, they mask the value of $k$.

**Phase 3: Signature Assembly**

- **Rounds 7-9:** With $R$ known, the value $r = R_x \pmod n$ is fixed. Parties now use their shares of $k \cdot x$ (computed via MtA) and the message $m$ to compute partial signatures $s_i$. Parties broadcast $s_i$ and locally sum them to obtain $s = \sum s_i$.

- **Verification:** Each party locally checks if $(r, s)$ verifies against the public key $Q$.

### 4.4 Limitations of GG18

While functional, GG18 suffered from two major operational drawbacks:

- **Anonymous Abort:** If the final signature verification failed (e.g., because a malicious party injected a bad value in Round 2), the protocol would simply output "Error." It offered no mathematical way to identify who cheated. This allowed a single malicious node to perform a Denial-of-Service (DoS) attack on the signing ceremony indefinitely.[4]

- **Range Proof Vulnerability:** The original paper's specifications for the ZK Range Proofs in the MtA step were insufficient. They did not enforce strict enough bounds, allowing for the "Alpha-Rays" attack where attackers could extract private key bits via modular overflow (see Section 8).[3]

## 5. GG20: Identifiable Abort and Optimization

GG20 (Gennaro & Goldfeder, 2020) was a direct response to the limitations of GG18. It introduced the concept of Identifiable Abort—the ability to deanonymize a cheater upon protocol failure—and optimized the online phase of signing.

### 5.1 The "Identifiable Abort" Mechanism

The core innovation of GG20 is the "Blame Phase." In MPC, identifying a cheater usually requires revealing the secret inputs to check correctness, which defeats the purpose of the protocol. GG20 circumvents this by using an Optimistic Execution model combined with **Commitment Consistency Verification**.

**Commitment-Based Blame Protocol:**

In Round 1 (Preprocessing), each party $P_i$ commits to their ephemeral nonce and blinding values:
$$c_i = H(k_i, \gamma_i, \rho_i, r_i)$$
where $r_i$ is a random nonce for the commitment scheme.

**The Audit Process (Upon Failure):**

When the final signature $(r, s)$ fails verification, parties enter the Blame Phase:

1. **Commitment Opening:** Each party $P_i$ publishes their preimage:
   $$(k_i, \gamma_i, \rho_i, r_i)$$
   All parties can now verify:
   $$c_i \stackrel{?}{=} H(k_i, \gamma_i, \rho_i, r_i)$$

2. **Consistency Checks:** For every pair of parties $(P_i, P_j)$ that ran the MtA protocol:
   - Alice (in MtA role): Proves that $E_A(a)$ matches her committed $k_i$ or $\gamma_i$
   - Bob (in MtA role): Proves that his encrypted contribution and ZK proofs are consistent with the transcript
   - The check verifies:
     $$E_{i,j}(k_i \cdot \gamma_j) \stackrel{?}{\text{matches commitment}} V_{i,j}$$

3. **Accusation:** If a party's commitment opening doesn't match the published value, or if their ZK proofs are inconsistent with the opening, they are identified as the cheater.

**Mathematical Guarantees:**

- **Completeness:** If all parties are honest, the signature will verify in Round 7, and no Blame Phase is triggered
- **Soundness:** If a party cheated (e.g., used different values in different rounds), their commitment opening will necessarily be inconsistent with either:
  - The hash digest $c_i$ published in Round 1, or
  - The ZK proofs provided in Round 2-6
- **Public Verifiability:** Even a third party (blockchain, auditor) can verify the Blame Phase transcript and confirm which party cheated

**Blame Phase Communication Complexity:**

- **Transmission:** Each party must broadcast their preimage (3-4 scalar values + hash randomness) ≈ **160 bytes per party**
- **Storage:** To enable blame attribution, the system must store all Round 1-6 transcripts. For n=5 parties, this is roughly **50-100KB per signing session**
- **Computation:** Verifying all consistency checks requires ~100-200 hash operations and ~50-100 elliptic curve verifications per cheating claim

**Optimistic Flow:** The protocol assumes everyone is honest. Parties execute the signing steps.

**Failure Trigger:** If the final signature $(r, s)$ is invalid, the protocol enters the Blame Phase.

**The Audit:** Since the signing failed, the "signature" produced is garbage and leaks no information about the true private key (assuming the failure happened in specific ways). Therefore, parties can safely "open" their commitments from the earlier rounds.

**Verification:** Parties broadcast the random nonces and blinding factors they used in the MtA phase. Honest parties verify these against the initial commitments (Round 1).

**Attribution:** The cheater is identified as the party whose revealed values do not mathematically align with their commitments or the ZK proofs provided. This mechanism acts as a strong deterrent against malicious DoS attacks.[4]

### 5.2 Protocol Restructuring: Offline vs. Online

GG20 optimized the user experience by splitting the signing process into Offline (Preprocessing) and Online phases.

**Total Rounds:** 7 (6 Offline + 1 Online).

**GG20 Offline/Online Architecture:**

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/mpc/4.png" class="img-fluid rounded z-depth-1" zoomable=true %}
    </div>
</div>

**Performance Impact:**

| Metric | GG18 | GG20 |
|--------|------|------|
| **User-Perceived Latency** | 9 rounds (~8-10s) | 1 round (~100-500ms) | 
| **Total Protocol Rounds** | 9 | 7 (6 offline + 1 online) |
| **Preprocessing Time** | N/A (no split) | 10-30 seconds (runs in background) |
| **Presignature Storage** | N/A | ~1KB per presignature |
| **Throughput Improvement** | Baseline | ~50-100x for user-facing latency |

**Practical Deployment:**

The offline/online split enables MPC wallets to achieve:
1. **Responsive User Experience:** Users click "Send" and receive confirmation in <1 second
2. **Load Balancing:** Heavy MtA computation distributed across idle time
3. **Buffer Management:** Systems can pre-generate 100-1000 presignatures in advance, handling traffic spikes without delay

**Preprocessing (Rounds 1-6):**

Parties perform the heavy lifting: the MtA exchanges, the Paillier encryptions, and the generation of the signature "structure" (the values $R$, $k$, and the additive shares of the product). Crucially, this phase is message-independent. It can be run hours or days before the user actually requests a transaction. The output of this phase is a "Presignature" object stored in the database.

**Online Signing (Round 7):**

When the message $m$ is received (e.g., a user clicks "Send" in a wallet), parties retrieve the Presignature. They perform a single local scalar multiplication involving $m$ and their presignature shares. They broadcast the resulting partial signature $s_i$. Once a party receives $t+1$ shares, they sum them to reconstruct the valid signature.

**Impact:** This reduced the effective latency for the user from ~9 round-trips to 1 round-trip, significantly improving the responsiveness of MPC wallets.[2, 5]

### 5.3 Mitigation of Paillier Attacks

GG20 introduced stricter ZK Range Proofs for the MtA phase. Specifically, it required proofs that the encrypted values were small enough to avoid wrapping around the modulus $N$. However, the complexity of implementing these proofs correctly still left room for implementation errors, leading to the eventual development of CGGMP21.

## 6. CGGMP21: Universal Composability and Proactive Security

CGGMP21 (Canetti, Gennaro, Goldfeder, Makriyannis, Peled 2021) represents the maturation of threshold ECDSA into a rigorously provable standard. While GG18 and GG20 were secure in standalone models, real-world wallets run multiple concurrent signing sessions, often interleaved with key generation or key refreshes. This concurrency opens up subtle attack vectors not covered by static security models.

### 6.1 Universal Composability (UC)

CGGMP21 is the first threshold ECDSA protocol proven secure in the UC Framework.

- **Definition:** UC security guarantees that a protocol remains secure even when it is composed (run in parallel or sequence) with arbitrary other protocols.

- **Significance:** This is critical for institutional custody providers (like Fireblocks or Coinbase) where thousands of signing requests for different wallets might occur simultaneously on the same infrastructure. A lack of UC security could theoretically allow an attacker to use messages from one signing session to break the security of another.

- **Adaptive Security:** The protocol handles adaptive adversaries, meaning the attacker can choose which parties to corrupt during the execution of the protocol, rather than having to decide beforehand. This models real-world intrusions more accurately.[6, 7]

### 6.2 Key Refresh and Proactive Security

One of the most significant operational additions in CGGMP21 is a native Key Refresh Protocol.

**The Problem:** In a standard threshold scheme, if an attacker compromises Party A in January and Party B in June, they possess 2 shares. Over time, a "slow" attacker can eventually collect all $t+1$ shares needed to reconstruct the key (the "Mobile Adversary" problem).

**Key Refresh Mechanism Visualization:**

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/mpc/5.png" class="img-fluid rounded z-depth-1" zoomable=true %}
    </div>
</div>

**Mathematical Model:**

$$\text{Epoch } t: x = \sum_{i=1}^{n} x_i^{(t)}$$
$$\text{Epoch } t+1: x = \sum_{i=1}^{n} x_i^{(t+1)} = \sum_{i=1}^{n} (x_i^{(t)} + z_i^{(t)}) \text{ where } \sum z_i^{(t)} = 0$$

An attacker must compromise at least $t+1$ parties **within the same epoch** to extract the key. Cross-epoch shares are uncorrelated.

**The Solution:** The Key Refresh protocol allows parties to generate a new set of shares $(x'_1, x'_2,..., x'_n)$ for the same underlying private key $x$.

$$x = \sum x_i = \sum x'_i$$

The old shares $x_i$ become useless. To compromise the key, an attacker must compromise $t+1$ parties within the same time epoch.

**Mechanism:** Parties generate random zero-sum shares $z_i$ such that $\sum z_i = 0$. Each party updates their share: $x'_i = x_i + z_i$.

**Encryption Upgrade:** CGGMP21 utilizes Diffie-Hellman (ElGamal) style encryption for the refresh steps where possible, avoiding the computationally expensive Paillier operations used in previous iterations. This makes the refresh process lightweight and suitable for frequent execution (e.g., every 15 minutes).[2]

**Practical Security Implications:**

| Scenario | GG18/GG20 | CGGMP21 with Key Refresh |
|----------|-----------|------------------------|
| Attacker compromises 1 party/month × 3 months | ✗ Full key stolen | ✓ Keys refresh faster than compromise rate |
| Attacker compromises 2 parties simultaneously | ✗ Vulnerable (if t=1) | ✓ Protected (requires same epoch) |
| Long-term institutional security (5+ years) | ❌ Accumulating breach risk | ✅ Breach window limited to refresh cycle |
| Compliance with key rotation mandates | ❌ Manual, error-prone | ✅ Automatic, enforced by protocol |

### 6.3 4-Round Architecture

CGGMP21 further optimizes the round complexity.

- **Rounds 1-3 (Preprocessing):** Handling the MtA conversion, nonce generation, and the "Cheap Accountability" checks.

- **Round 4 (Online):** Non-interactive signature generation.

- **Cheap Accountability:** Unlike GG20, which required a heavy "Blame Phase" post-failure, CGGMP21 integrates verification steps into the preprocessing. If a party cheats during the MtA phase, they are detected before the online phase begins. This avoids the need to store massive transcripts for potential blame assignment, reducing storage overhead.[7]

## 7. MPC-CMP: Industrial Optimization and Cold Storage

MPC-CMP is a protocol developed by the Fireblocks cryptography team (Canetti, Makriyannis, Peled) that builds upon the theoretical foundations of CGGMP21 but optimizes specifically for the constraints of high-frequency trading and cold storage security.

### 7.1 The 1-Round Signing Breakthrough

While GG20 and CGGMP21 support a 1-round online phase, MPC-CMP is architected to treat the preprocessing phase as a continuous background process.

- **Pre-Processing Buffer:** An MPC-CMP system continuously generates pre-signatures (tuples of $k, r$, and auxiliary data) during idle time. These are stored in a secure queue.

- **Instant Signing:** When a transaction request is authorized, the system pops a pre-signature from the buffer. The signing parties perform one local scalar multiplication and broadcast the result.

- **Performance:** This architecture allows for signing speeds up to 800% faster than GG18. The perceived latency for the user is just the network propagation time of a single packet.[8]

### 7.2 Enabling Air-Gapped MPC (Cold Storage)

Historically, MPC was considered incompatible with "Cold Storage" (offline devices) because of the requirement for multiple rounds of interactive communication (the "chattiness" of the MtA protocol). Connecting a cold device to a network to perform 9 rounds of handshake defeats the purpose of air-gapping.

MPC-CMP solves this via its non-interactive signing capability:

**Air-Gapped Signing Workflow (QR Code Transfer):**

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/mpc/6.png" class="img-fluid rounded z-depth-1" zoomable=true %}
    </div>
</div>

**Security Properties of Air-Gapped MPC:**

The ability to perform threshold signing without network connectivity to the cold device provides several critical security advantages:

| Aspect | Traditional Multisig | Air-Gapped MPC-CMP |
|--------|---------------------|-------------------|
| **Network Exposure** | Requires network connection | No network connection to signer |
| **Attack Surface** | Network exploits, DDoS, MITM | Only QR code camera vulnerability |
| **Key Reconstruction** | Never reconstructed | Never reconstructed |
| **Compliance** | Meets cold storage mandate | Native cold storage support |
| **Latency** | 2-5 network round trips | ~30-60 seconds user workflow |
| **Regulatory** | ✓ Acceptable but limited | ✓ Exceeds regulatory requirements |

**Implementation Details:**

1. **Online Aggregator:** The connected server (hot wallet) prepares the transaction and selects a pre-signature from the buffer.

2. **Unidirectional Transfer:** The server encodes the transaction hash and the pre-signature metadata into a QR code.

3. **Offline Signing:** The user scans the QR code with an air-gapped signing device (e.g., a dedicated tablet or hardware wallet). The device holds one key share. It computes its signature share $s_i$ offline using the pre-signature data.

4. **Return Transfer:** The device displays the signature share as a QR code, which is scanned back by the online server.

5. **Reconstruction:** The server aggregates the share with others to broadcast the transaction.

**Practical Benefits:**

- **Regulatory Compliance:** Exceeds SEC/SOX requirements for cold storage
- **Insurance Underwriting:** Insurance carriers offer higher coverage for cold storage MPC
- **User Confidence:** Private key never touches the internet
- **Operational Ease:** No special hardware required beyond a second device
- **Auditability:** QR code transfers create a cryptographic audit trail

This workflow makes MPC viable for regulatory environments that mandate cold storage, previously the domain of MultiSig or physical HSMs.[9, 8]

## 8. DHSMPC & nQSMax: Directed Computation and Post-Quantum Security (2025-2026)

As institutional adoption of MPC protocols accelerates, two critical limitations of MPC-CMP have emerged: signature-only output constraints and vulnerability to quantum threats. DHSMPC (Directed Hidden State Multi-Party Computation) and nQSMax (its quantum-safe variant) address these limitations by enabling arbitrary computational graphs and post-quantum cryptographic foundations.

### 8.1 Four Limitations of MPC-CMP That Drive DHSMPC Development

1. **Signature-Only Output:** MPC-CMP is architecturally designed for threshold ECDSA signing only. It cannot compute arbitrary functions. This restricts use to traditional custody and payment scenarios, missing emerging opportunities in:
   - Federated machine learning (hospitals sharing diagnostic models without exposing patient data)
   - Decentralized finance (liquidation decisions based on hidden state computation)
   - Privacy-preserving auctions (winner determination without revealing bids)

2. **Lack of AI/ML Integration:** The protocol does not accommodate neural network weights or continuous computational states. Training or inference on encrypted data across parties requires entirely different machinery (e.g., secure aggregation for federated learning, which MPC-CMP does not natively support).

3. **Fixed Computation Patterns:** MPC-CMP assumes a fixed signing ceremony repeated at scale. It cannot adapt to variable-complexity computations. A computation that requires 4 rounds and another requiring 2 rounds use the same 4-round framework, wasting communication for the simpler computation.

4. **No Quantum Resistance:** ECDSA and all current threshold protocols rely on integer factorization and discrete logarithm hardness. Cryptographically Relevant Quantum Computers (CRQCs) could factor 2048-bit RSA moduli in ~8 hours by 2030-2040. An attacker using "harvest now, decrypt later" attacks could collect today's transactions and decrypt them post-quantum.

### 8.2 DHSMPC: Hidden State, Directed Execution, and Variable-Round Optimization

DHSMPC extends MPC beyond key management to arbitrary stateful computation via three core innovations:

**Hidden State Abstraction:**
- Traditional MPC operates on keys (256-bit scalar values)
- DHSMPC abstracts "cryptographic states" to include neural network weights, model parameters, and computational intermediates
- Example: A federated learning consortium can jointly train a diagnostic model (~1-10MB of state) across hospital servers without exposing raw patient data
- State size: ~1MB typical (1000x larger than keys, requiring adapted protocols)

**Computation Graph (DAG) Representation:**

DHSMPC accepts a computation as a **Directed Acyclic Graph (DAG)** rather than a fixed protocol:

$$\mathcal{G} = (V, E, \text{ops})$$

where:
- **Vertices V:** Computational operations (addition, multiplication, ReLU, matrix multiply, etc.)
- **Edges E:** Data flow between operations (output of one becomes input to another)
- **Operations ops:** Each vertex is labeled with its operation type and complexity

**Example DAG for "Liquidation Decision":**

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/mpc/13.png" class="img-fluid rounded z-depth-1" zoomable=true %}
    </div>
</div>

**Round Optimization Algorithm:**

DHSMPC analyzes the DAG to determine the minimum number of rounds needed:

1. **Compute DAG Depth:** The longest path from input to output determines the minimum rounds needed
   - Linear dependencies (A → B → C): depth = 3, requires 3 round "phases"
   - Fully parallel operations (A, B, C independent): depth = 1, requires 1 round phase
   
2. **Group Operations:** DHSMPC groups operations by dependency level
   - **Level 0:** All input shares (no communication needed)
   - **Level 1:** All operations dependent only on Level 0 (1 communication round)
   - **Level 2:** All operations dependent only on Level 0-1 (2nd communication round)
   - ... and so forth

3. **Estimate Rounds:** For the liquidation example:
   - Level 0 (inputs): No communication
   - Level 1 (comparison operations in parallel): 2 MtA rounds → ~2 network rounds
   - Level 2 (AND gate): 1 addition round → ~1 network round
   - **Total: 3 rounds** (vs. fixed 4-6 rounds in traditional MPC-CMP)

**Mathematical Formulation:**

Let $\text{depth}(\mathcal{G})$ be the longest path in the DAG:

$$\text{rounds}_{\text{DHSMPC}}(\mathcal{G}) = \text{depth}(\mathcal{G}) + C$$

where $C$ is a small constant overhead (~1-2 rounds) for setup and aggregation. For a computation with:
- **Highly parallel graph:** rounds ≈ 2-3
- **Linear dependencies:** rounds ≈ depth(G) + 2
- **Deep sequential chain:** rounds ≈ 5-7 (worst case)

**Directed Execution:**
- MPC-CMP uses fixed signing templates
- DHSMPC accepts a computation graph as input: DAG specifying operations (additions, multiplications, matrix operations), dependencies, and data flow
- The protocol optimizes round complexity based on graph structure (e.g., fully parallel operations reduce rounds; sequential chains increase rounds)
- Example: A decentralized liquidation engine can compute "liquidate if collateral_ratio < 0.6 AND price_change > -20%" with custom logic, not just signatures

**Variable-Round Optimization:**
- DHSMPC analyzes the computation graph at setup time
- For highly parallel graphs (many independent operations), it uses 2-3 rounds
- For sequential chains (strict dependencies), it uses 4-6 rounds
- Parties avoid paying communication penalty for non-existent dependencies
- Result: Reduces median latency by ~40% compared to fixed-round protocols

**Computational Complexity Comparison:**

| Computation Type | MPC-CMP Rounds | DHSMPC Rounds | Latency Reduction |
|-----------------|--|--|--|
| Simple comparison (1 MtA) | 4 | 1-2 | 50-75% |
| Federated learning grad (100 multiplies, parallel) | 4 | 2 | 50% |
| Deep neural network (50 layers sequential) | 4 | 6-8 | -50% (longer due to depth) |
| Auction winner determination (n parties, n comparisons) | 4 | 3-4 | 0-25% |

### 8.3 nQSMax: Quantum-Safe Variant

nQSMax replaces classical cryptographic hardness assumptions with lattice-based cryptography (Learning With Errors, LWE), which remains hard against quantum computers.

**Post-Quantum Threat Timeline and Migration Strategy:**

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/mpc/7.png" class="img-fluid rounded z-depth-1" zoomable=true %}
    </div>
</div>

**Cryptographic Substitutions:**

| Component | Classical | Quantum-Safe | Key Sizes | Impact |
|-----------|-----------|--------------|-----------|--------|
| Encryption | Paillier (2KB keys) | Kyber (1KB keys) | ~50% reduction | Faster broadcasts |
| Signatures | ECDSA (256-bit) | Dilithium (2560-bit) | ~10x larger | Acceptable for post-quantum |
| Handshake | DH key exchange | Kyber-based exchange | New protocol | No patents on LWE |

**Quantum Threat Timeline:**
- **2030-2040:** CRQCs feasible (estimated 10-20M physical qubits)
- **Harvest-Now-Decrypt-Later (HNDL):** Adversaries collecting ECDSA signatures today can decrypt private keys in 2030+
- **Regulatory Mandate:** SEC/NIST expects mandatory post-quantum transition by end of 2028
- **Full Migration:** Industry-wide migration target: 2035

**Key Size Growth:**
- Kyber public key: ~1184 bytes (vs. Paillier modulus: 2048 bytes)
- Dilithium signature: 2420 bytes (vs. ECDSA: 64 bytes)
- Total protocol size increase: ~3x (acceptable given quantum resilience)

### 8.4 Empirical Characteristics and Use Cases

**Performance Comparison (7-Protocol Matrix):**

| Protocol | Era | Signing Rounds | Output Type | Quantum Resistant | Max Parties | Latency |
|----------|-----|---|------|---|--|---|
| Shamir's SSS | Traditional | Unlimited | Key recovery only | No | 1000+ | 500-800ms |
| GG18 | 2018-2020 | 9 | ECDSA signature | No | 15-20 | 8-10s |
| GG20 | 2018-2020 | 7 | ECDSA signature | No | 15-20 | 6-8s |
| CGGMP21 | 2021-2024 | 4 | ECDSA signature | No | 15-20 | 3-4s |
| MPC-CMP | 2021-2024 | 1 (online) | ECDSA signature | No | 15-20 | <1s |
| DHSMPC | 2025-2026 | 2-6 (variable) | Arbitrary function | No | 100-1000 | 1-3s |
| nQSMax | 2025-2026 | 2-6 (variable) | Arbitrary function | Yes | 100-1000 | 2-4s |

**Real-World Use Cases:**

1. **Federated Learning (Healthcare Consortium):**
   - 10 hospitals jointly train a diagnostic model without sharing raw patient records
   - DHSMPC aggregates gradient updates encrypted
   - Each hospital's data remains encrypted during training
   - Model converges in 20-50 rounds of gradient descent (~30-60 minutes vs. centralized 1 hour, acceptable tradeoff for privacy)

2. **DeFi Liquidation (Decentralized Refinancing):**
   - Liquidation decision: IF (collateral_value < 1.5 * debt) AND (daily_volatility > 0.15) THEN liquidate
   - Computation graph: 2 multiplications + 1 comparison (2 rounds in DHSMPC)
   - Parties never see each other's collateral values; only the liquidation signal is revealed
   - Prevents information leakage to competitors, enables fair liquidation mechanics

3. **Privacy-Preserving Auction:**
   - $n$ parties bid encrypted amounts
   - Auction house runs DHSMPC to compute (1) winner, (2) price clearing, (3) refunds
   - All computation hidden; only outcomes revealed
   - Prevents bid sniping, front-running, and collusion detection

4. **Post-Quantum Transition (Hybrid Keys):**
   - Current: ECDSA signatures stored on blockchain
   - 2028-2030: Rotate to hybrid keys (ECDSA + Dilithium)
   - 2035+: Pure Dilithium via nQSMax
   - Migration path: Gradual, not disruptive (can coexist for 5+ years)

### 8.5 Connection to FROST and Future Directions

While DHSMPC enables arbitrary computation, FROST (Flexible Round-Optimized Schnorr Threshold) offers an important complementary approach for blockchains supporting Schnorr signatures.

**FROST Advantages:**
- Linear signing equation ($s = k + ex$) requires no homomorphic encryption or MtA protocol
- Just 2 rounds total (optimizable to 1 with preprocessing)
- Native support for batch signing (multiple messages in parallel)
- Seamless integration with Schnorr aggregation and zero-knowledge proofs
- Ideal for Bitcoin Taproot, Solana, and other Schnorr-based systems

**FROST vs. DHSMPC Trade-offs:**
- FROST: Minimal rounds (2) but signature-only output
- DHSMPC: Variable rounds (2-6) but arbitrary computation + AI/ML support
- FROST + DHSMPC: Hybrid approach where FROST handles key management, DHSMPC handles computation if needed

## 9. Vulnerability Analysis: The Paillier Modulus Attacks

The progression from GG18 to CGGMP21 was significantly accelerated by the discovery of critical vulnerabilities in how the Paillier encryption scheme was implemented in the MtA phase. These attacks—Alpha-Rays and 6ix1een—demonstrated that theoretical security does not always translate to implementation security.

### 9.1 The Mechanism of the Attack

In the MtA phase, Alice sends $c = E_A(a)$ to Bob. Bob computes $c' = c^b \cdot E(\beta) = E(a \cdot b + \beta)$ and returns it.
The security relies on Bob adding enough "noise" ($\beta$) to mask the product $a \cdot b$. Furthermore, all operations happen modulo $N$ (Alice's Paillier modulus).

The attacks exploit modular overflow and malformed moduli:

- **Small Factors:** If Alice is malicious, she can generate a Paillier modulus $N$ that is not a product of two large primes, but contains small factors or has specific properties.

- **Overflow:** If Bob does not prove that his input $b$ and his noise $\beta$ are strictly within a safe range (much smaller than $N$), an attacker can manipulate the values such that the decryption $a \cdot b + \beta \pmod N$ wraps around the modulus in a predictable way.

- **Extraction:** By interacting with the victim (Bob) multiple times—roughly 16 signatures for the "6ix1een" attack—Alice can use the Chinese Remainder Theorem (CRT) to extract bits of Bob's secret $b$ from the residues of the modular overflows. Eventually, Alice reconstructs Bob's full private key share.[2, 3]

### 9.2 Mitigation via Zero-Knowledge Proofs

Protocol updates (patched GG18, GG20, and native CGGMP21) introduced mandatory, rigorous ZK proofs to counter this:

- **Modulus Proofs:** At Key Generation, every party must prove their Paillier modulus $N$ is square-free and biprime (a Paillier-Blum modulus).

- **Range Proofs:** During MtA, every encrypted value sent must be accompanied by a ZK proof demonstrating that the plaintext lies strictly within an interval $[0, q^3]$. This ensures that no overflow can occur during the homomorphic operations.

- **Ring-Pedersen Parameters:** CGGMP21 utilizes Ring-Pedersen parameters to make these range proofs more efficient and sound under the UC framework.[2, 6]

## 10. Comparative Analysis: Protocol Selection Matrix

The following table synthesizes the operational trade-offs between the four protocols.

| Feature | GG18 | GG20 | CGGMP21 | MPC-CMP |
|---------|------|------|---------|---------|
| Total Signing Rounds | 9 | 7 (6 Pre + 1 On) | 4 (3 Pre + 1 On) | 1 (Online)* |
| Identifiable Abort | No (Anonymous) | Yes (Blame Phase) | Yes (Cheap/Pre-check) | Yes |
| Security Model | Static | Rushing | UC / Adaptive | UC / Adaptive |
| Encryption | Paillier | Paillier | Paillier + ElGamal | Paillier Optimized |
| Key Refresh | Manual/External | Manual/External | Native (Proactive) | Native (Auto) |
| Cold Storage Capable | No | Partially | Yes | Native Support |
| Complexity | Moderate | High | Very High | Proprietary |

**Protocol Rounds Optimization Visualization:**

The progression of protocols demonstrates a relentless optimization in round complexity—the number of communication rounds required to produce a valid signature. Each reduction in rounds directly translates to reduced latency and improved user experience.

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/mpc/8.png" class="img-fluid rounded z-depth-1" zoomable=true %}
    </div>
</div>

**Key Innovation at Each Step:**
- **GG18 → GG20:** Identifiable abort eliminates anonymous DoS attacks; offline/online split reduces perceived latency
- **GG20 → CGGMP21:** UC-secure composition enables concurrent signing; native key refresh provides proactive security
- **CGGMP21 → MPC-CMP:** Pre-signature buffering and non-interactive signing enables instant authorization; air-gapped support achieves cold storage compatibility

### Key Takeaways for Implementation:

- **Legacy Systems:** GG18 is largely considered obsolete for new deployments due to the lack of identifiable abort and the complexity of patching the Paillier vulnerabilities correctly.

- **Academic Standard:** CGGMP21 is the current recommendation for open-source libraries (e.g., Safeheron, ZenGo) due to its UC security proofs.

- **Enterprise Performance:** MPC-CMP is the preferred architecture for high-volume custodians where millisecond latency and air-gapped security are business-critical requirements.

- **Emerging Capability:** DHSMPC/nQSMax represents the frontier for AI/ML integration and quantum-safe threshold computation, target adoption 2025-2027.

## 11. Comparative Architectures and Future Directions

While Threshold ECDSA solves the immediate problem of securing Bitcoin/Ethereum keys, the broader landscape of privacy-preserving computation offers alternative and complementary approaches.

### 11.1 FROST: The Schnorr Alternative

FROST (Flexible Round-Optimized Schnorr Threshold) is the leading protocol for blockchains that support Schnorr signatures (e.g., Bitcoin Taproot, Solana).

- **Linearity Advantage:** Unlike ECDSA, Schnorr signatures are linear ($s = k + ex$). Aggregating shares is a simple addition operation. There is no need for Paillier encryption or the complex MtA protocol.

- **Performance:** FROST requires only 2 rounds (optimizable to 1 with preprocessing) and is computationally lightweight.

- **Unidentifiability Risk:** Decentralized implementations of FROST must be careful. Without a central aggregator, a malicious party can send inconsistent nonce commitments to different peers, partitioning the network and causing honest parties to blame each other. Mitigations involve an extra round of commitment consistency checks.[10, 11]

### 11.2 MPC + TEE (Hybrid Architectures): Defense-in-Depth Strategy

Emerging architectures (e.g., Phala Network, Fairblock, Fireblocks SGX) combine software MPC with hardware Trusted Execution Environments (TEEs) like Intel SGX, Intel TDX, or AWS Nitro. This hybrid approach stacks mathematical security (MPC) with hardware isolation, creating **Defense-in-Depth** against both cryptographic and physical attacks.

**The Core Concept:**

Instead of running the MPC protocol in the standard OS memory (vulnerable to root access, memory scrapers, cold boot attacks), the key shares and signing logic are loaded into an **encrypted hardware enclave**:

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/mpc/9.png" class="img-fluid rounded z-depth-1" zoomable=true %}
    </div>
</div>

**Intel SGX (Software Guard Extensions) - Deprecated but Educational:**

Intel SGX was introduced in 2013 but has been largely deprecated due to vulnerabilities. However, it's educational for understanding TEE principles:

- **Enclave Concept:** Small protected memory region (24-128 MB typical) that runs encrypted
- **Memory Encryption:** Data in enclave memory is encrypted with a key held only by the CPU
- **Isolation:** Even root-privileged OS cannot access enclave memory; the CPU enforces bounds-checking
- **Limitations Discovered:**
  - Spectre/Meltdown attacks could leak enclave data via cache side-channels
  - Timing attacks exploitable by measuring execution time differences
  - PageTable attacks via page-level access patterns
  - No longer recommended for high-security MPC key management

**Intel TDX (Trust Domain Extensions) - Modern Successor:**

TDX is Intel's replacement for SGX, addressing previous vulnerabilities:

- **Hardware-Level Encryption:** Entire guest VM memory encrypted at hardware level
- **Isolation:** Even hypervisor cannot access TD (Trusted Domain) memory
- **Large Memory Support:** Supports gigabytes of protected memory (vs. 128MB for SGX)
- **Attestation:** Provides cryptographic proof that code running in TD is unmodified
- **Protection Against:**
  - Hypervisor attacks (hypervisor code cannot read/modify TD memory even with full privileges)
  - Side-channel attacks (mitigated but not eliminated via constant-time execution)
  - Cold boot attacks (memory is encrypted with key tied to hardware)

**AWS Nitro Enclaves - Cloud-Native TEE:**

AWS Nitro provides container-based isolation for cloud MPC deployments:

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/mpc/10.png" class="img-fluid rounded z-depth-1" zoomable=true %}
    </div>
</div>

- **Container Isolation:** Nitro Enclave runs in a separate virtual machine with 0 access to parent EC2 instance memory
- **Attestation Document:** Enclave provides signed attestation proving:
  - Enclave image hash (PCR - Platform Configuration Register)
  - Timestamp when attestation was issued
  - AWS account ID and region
- **Use Case:** MPC key shares stored in Nitro, signing requests authenticated via KMS
- **Cost:** Per-second pricing (~$0.01/second) suitable for high-frequency signing
- **Advantage:** No special hardware required; works on standard AWS infrastructure

**Hybrid MPC + TEE Architecture:**

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/mpc/11.png" class="img-fluid rounded z-depth-1" zoomable=true %}
    </div>
</div>

**Blockchain and AI Use Cases for Hybrid MPC + TEE:**

The combination of MPC and TEE creates powerful new capabilities across blockchain and artificial intelligence applications:

**Blockchain Applications:**

1. **Cross-Chain Bridge Security:**
   - **Use Case:** A multi-signature validator set for bridge operations between Ethereum, Solana, and Polygon
   - **Architecture:** Each validator's key share held in a TEE; threshold MPC ensures no single validator can sign fraudulent bridge transactions
   - **Benefit:** Combines TEE isolation (prevents OS-level key extraction) with threshold cryptography (prevents single-validator theft)
   - **Attack Resistance:** Even if an attacker compromises one validator's TEE, they cannot forge bridge signatures without t additional validators
   - **Example:** Stargate Finance 3-of-5 multi-sig across TEE-hardened validators protecting $2B+ in bridged assets

2. **Decentralized Exchange (DEX) Settlement:**
   - **Protocol Flow:**
     - User submits swap order → DEX collects orders → settlement contract
     - Settlement agents run MPC in TEEs to compute clearing prices and final transfers
     - Each agent's TEE proves (via attestation) that the settlement algorithm was executed correctly
   - **Privacy Guarantee:** Individual orders remain encrypted during settlement computation
   - **Example:** dYdX Protocol settlement with privacy and speed via MPC + TEE

3. **Governance and DAO Treasury:**
   - **Use Case:** Protecting DAO treasury keys where signers are geographically distributed
   - **Architecture:** Each DAO signer holds a key share in their local TEE; threshold MPC-CMP enables signing without reconstructing the private key
   - **Compliance:** Attestation logs provide immutable audit trail for regulatory requirements
   - **Multi-Layer Governance:** Smart contracts can verify attestation proofs before allowing treasury transactions

**AI/ML Applications:**

1. **Federated Learning with Privacy Preservation:**
   - **Use Case:** Training diagnostic AI models across hospitals without exposing patient data
   - **Architecture:**
     - Each hospital: Trains local model, extracts gradients, encrypts with MPC shares
     - Aggregation server: Runs secure aggregation in a TEE to combine encrypted gradients
     - Server's attestation proves: (a) no plaintext access to gradients, (b) correct aggregation algorithm
   - **Convergence:** Model converges to same accuracy as centralized training (100-500 gradient rounds), but with complete privacy
   - **Regulatory:** HIPAA-compliant due to lack of data centralization and encrypted-gradient processing
   - **Quantitative Impact:** Medical imaging consortium of 10 hospitals trains COVID-19 detection model to 95% accuracy without sharing any raw images

2. **Confidential Machine Learning Inference:**
   - **Use Case:** Healthcare provider runs inference on encrypted patient data without seeing the data
   - **Architecture:**
     - Patient: Encrypts medical records with MPC shares across two providers
     - Provider A: Runs inference on encrypted data in TEE, shares result via threshold MPC
     - Provider B: Runs inference verification (MPC threshold ensures correctness)
   - **Output:** Encrypted prediction returned to patient; patient decrypts locally
   - **Security Properties:** Neither Provider A nor Provider B can see the patient's data or the model's weights (if model is also encrypted)

3. **Collaborative AI Model Training:**
   - **Use Case:** Competitors in same industry (insurance, banking) jointly train risk models to improve accuracy without sharing proprietary data
   - **Protocol:** Each party contributes private dataset to MPC aggregation; training happens in a consortium's shared TEE infrastructure
   - **Example:** Insurance companies training fraud-detection models collaboratively; no company sees another's claim data
   - **Economics:** Time to model convergence reduced by 70% vs. individual model training due to larger effective dataset

4. **Secure Neural Network Verification:**
   - **Use Case:** Verifying that a third-party AI model (e.g., autonomous driving neural network) is unbiased and meets safety requirements
   - **Architecture:** Model weights encrypted via MPC; auditors run verification proofs in shared TEE without seeing the model
   - **Regulatory:** Certification agencies use MPC + TEE to audit proprietary AI models for compliance with fairness/safety regulations
   - **Application:** FDA certification of medical AI, autonomous vehicle safety validation

**Security Properties of Hybrid MPC + TEE:**

| Threat Model | MPC Alone | TEE Alone | MPC + TEE |
|---|---|---|---|
| Compromised OS on one party | ✓ Protected | ✗ Exposed | ✓ Protected |
| Memory scraper attack | ✓ Protected | ✗ Exploitable | ✓ Protected |
| Hypervisor breach (cloud) | ✓ Protected | ✗ Exposed | ✓ Protected |
| Physical memory bus attack | ✗ Vulnerable | ✓ Encrypted | ✓ Protected |
| Side-channel timing attacks | ✗ Vulnerable | ~ Mitigated | ~ Mitigated |
| Malicious threshold parties | ✓ Protected (MPC bounds) | Irrelevant | ✓ Protected |

**Attestation for Accountability:**

Remote attestation enables a critical property: **accountability without full transparency**

- **Problem:** In decentralized MPC, if a party misbehaves, how do others prove it to a blockchain?
- **Solution:** TEE provides a signed attestation of:
  1. The exact code running inside the enclave (cryptographic hash)
  2. The timestamp and signer identity
  3. A signature from the hardware manufacturer (Intel, AWS, etc.)
- **Blockchain Integration:** Smart contract verifies the attestation, confirming that the misbehaving code was indeed the claimed party's software. This solves the "Sybil" attack problem where a single entity runs multiple virtual identities.

**Trade-offs of Hybrid Approach:**

| Aspect | Benefit | Drawback |
|---|---|---|
| **Security** | Combines math + hardware trust | Adds trust in hardware vendor |
| **Latency** | TEE reduces network rounds needed | Enclave-exit latency ~10-100μs |
| **Cost** | Fewer communication rounds | Hardware attestation per signature |
| **Regulatory** | Auditable via attestation logs | Vendor lock-in (AWS, Intel, etc.) |
| **Scalability** | Higher party counts feasible | Limited enclave memory (TDX: ~10GB) |

**Practical Deployment (Fireblocks Example):**

Fireblocks integrates MPC with hardware enclaves:
- Customer's key share stored in Nitro Enclave (isolated, encrypted)
- Fireblocks' key share stored on their HSM or TEE
- 2-of-2 threshold across two different infrastructure providers
- Attestation logs provide audit trail for compliance
- Result: Even if Fireblocks' infrastructure is compromised, attacker cannot forge signatures without customer's enclave

### 11.3 Federated Learning and Secure Aggregation

The principles of MPC are extending into Privacy-Preserving Machine Learning (PPML).

- **Secure Aggregation:** In Federated Learning, a server aggregates model updates (gradients) from thousands of client devices. This is mathematically analogous to aggregating signature shares. Protocols like Bonawitz et al. (used by Google) utilize Shamir Secret Sharing to aggregate high-dimensional vectors.

- **SMPAI (Secure Multi-Party AI):** Recent research by J.P. Morgan combines MPC with Differential Privacy (DP). While MPC protects the individual inputs during aggregation, DP adds noise to the final result to prevent the trained model from leaking information about the training data (a "Model Inversion" attack). This represents the convergence of MPC with AI safety.[14, 15]

## 12. Operationalizing MPC: Best Practices

Implementing these protocols in a production environment requires more than just correct code.

- **The Ceremony:** Key generation should be treated as a "Ceremony." For high-value keys, this involves air-gapped machines, Faraday cages, and the destruction of the ephemeral hardware used for the setup.[16]

- **Disaster Recovery (DR):** If a user loses their key share, they lose their funds. Implementations often use a "Backup Share" or a "Social Recovery" mechanism where a threshold of trusted friends or institutions can help reconstruct the user's share (without ever reconstructing the full key).

- **GDPR Compliance:** A critical legal distinction exists between Anonymization and Pseudonymization.
  - *Anonymization* (irreversible removal of identifiers) exempts data from GDPR.
  - *Pseudonymization* (replacing names with IDs/Keys) does not exempt data from GDPR.
  - MPC shares are typically considered pseudonymous because they can theoretically be recombined to identify the key holder. Therefore, MPC operators must still comply with data protection regulations.[17, 18]

## 13. Conclusion

The evolution of Threshold ECDSA from GG18 to MPC-CMP illustrates the rapid maturation of cryptographic infrastructure. What began as a theoretical feasibility proof has evolved into a robust, industrial-grade standard capable of securing the global digital asset economy.

GG18 broke the barrier, proving trustless setup was possible. GG20 introduced the accountability necessary for decentralized trust. CGGMP21 provided the formal security proofs (UC) required for high-concurrency environments. MPC-CMP optimized these foundations for the speed and air-gapped security demanded by institutional finance.

As the industry moves forward, the integration of these protocols with TEEs offers a promising "Defense in Depth" strategy, while the migration of blockchains to Schnorr/FROST may eventually reduce the computational complexity of custody. However, for the foreseeable future, the secure implementation of ECDSA via CGGMP21/MPC-CMP remains the cornerstone of digital asset security.

## References

[1] Gennaro & Goldfeder, 2018 (GG18 Paper).

[2] Comparative Examination of Threshold ECDSA.

[3] Makriyannis & Peled, "A Note on the Security of GG18".

[4] Gennaro & Goldfeder, 2020 (GG20 Paper).

[5] Howard Tam, "GG20 Technical Notes".

[6] Canetti et al., 2021 (CGGMP21 Paper).

[7] Safeheron, "Multi-party-sig-cpp and CGGMP21".

[8] Fireblocks MPC-CMP Technical Blog.

[9] Fireblocks, "GG18 and GG20 Paillier Key Vulnerability".

[13] Fairblock, "Accountable MPC using TEEs".

[17] Decentriq, "Anonymization vs Pseudonymization".
