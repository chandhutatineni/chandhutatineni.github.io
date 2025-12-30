---
layout: post
title: Analysis of Privacy-Preserving Machine Learning in the Age of Sensitive Data
date: 2025-08-10
description: Training models on Sensitive data, approaches, tradeoffs, and recommendations.
tags: AI/ML
categories: AI/ML
related_posts: false
thumbnail: assets/img/ppml/ppml.png
---

## **1\. Introduction: The Data-Utility Friction in Modern AI**

The contemporary artificial intelligence ecosystem operates on a fundamental and increasingly precarious paradox: the efficacy of machine learning (ML) models is inextricably linked to the granularity, volume, and fidelity of the data upon which they are trained, yet the acquisition, aggregation, and processing of this data increasingly contravene established ethical norms, legal frameworks, and societal expectations regarding individual privacy. We stand at a critical juncture in the evolution of computational intelligence where the "memorization" capacity of deep neural networks—the very feature that enables high-performance predictive capabilities in complex domains like natural language processing and medical diagnostics—simultaneously constitutes a severe, systemic vulnerability.

Deep neural networks (DNNs), characterized by their millions or even billions of parameters, have demonstrated an uncanny and often unintended ability to memorize training data. While the objective of training is for these models to learn generalized patterns—the syntax of a language, the visual features of a malignant tumor, or the purchasing propensity of a demographic—they frequently encode specific, idiosyncratic details of individual training examples within their weight matrices. When this training data comprises sensitive information, such as electronic health records (EHR), financial transaction histories, biometric data, or private communications, the model itself becomes a vector for data leakage. The assumption that a model is merely a functional abstraction of data, distinct and safe from the raw inputs, has been proven dangerously false.

The regulatory landscape has shifted dramatically to address this reality. Frameworks such as the European Union's General Data Protection Regulation (GDPR), the California Consumer Privacy Act (CCPA), and the Health Insurance Portability and Accountability Act (HIPAA) in the United States impose rigorous standards on data handling.3 However, these regulations primarily target the storage and transmission of personal data, often leaving the status of machine learning models ambiguous. A critical debate has emerged within legal and technical circles: if a model allows for the reconstruction of training data, does the model itself constitute "personal data" under the law? Recent research suggests that the process of turning training data into machine-learned systems is not unidirectional; information flows can be reversed, turning the model into a database of its own training set.

This report provides an exhaustive, expert-level examination of the privacy landscape in machine learning. It begins by dissecting the vulnerabilities inherent in standard model training, explicitly detailing the sophisticated adversarial attacks that exploit these weaknesses to extract sensitive information. It then critiques traditional data-layer interventions, demonstrating their mathematical insufficiency in the face of high-dimensional data. Finally, it progresses through the hierarchy of Privacy-Preserving Machine Learning (PPML) techniques—distinguishing between *Confidentiality* (hiding the data during training) and *Privacy* (hiding the data in the result)—to determine a "best-in-class" hybrid approach for securing the future of AI.

## **2\. The Problem Space: Vulnerabilities in the Training Lifecycle**

To understand the necessity of Privacy-Preserving Machine Learning (PPML), one must first rigorously define the threat landscape. The traditional view of a machine learning model as a "black box" that insulates the training data from the end-user has been definitively shattered. Adversarial techniques have evolved to exploit the very mathematical principles that allow models to learn, turning optimization objectives into privacy vulnerabilities.

### **2.1 The Myth of the Black Box**

In standard machine learning pipelines, raw data is aggregated centrally, preprocessed, and fed into an optimization algorithm—typically a variant of Stochastic Gradient Descent (SGD)—to minimize a loss function. The resulting artifacts, the model weights and biases, are widely considered intellectual property. However, recent research indicates that these weights function as a compressed, albeit lossy, database of the training set. A privacy breach occurs if an adversary can use the model's output, or the model itself, to infer the values of unintended, sensitive attributes used as input.

The vulnerability stems from the fundamental mechanics of learning. A model updates its internal state to minimize the error on specific training examples. If a model is over-parameterized—having more parameters than training data points, which is common in deep learning—it can theoretically memorize the labels or features of specific outliers to achieve zero training error. This "overfitting" to the training distribution is the crack in the armor that privacy attacks exploit.

### **2.2 Taxonomy of Privacy Attacks**

The vulnerabilities of ML models are typically categorized based on the adversary’s goal and their access level (black-box vs. white-box).

#### **2.2.1 Membership Inference Attacks (MIA)**

Membership inference is often considered the "canary in the coal mine" for privacy leakage. In an MIA, an adversary attempts to determine whether a specific individual's data record was used to train a target model.6 This binary classification problem—"in" or "out"—may seem benign but carries severe implications. For instance, if an adversary can infer that a specific patient’s clinical record was used to train a model strictly associated with an HIV susceptibility study, the privacy breach is equivalent to revealing the patient’s diagnosis, regardless of the model's specific output for that patient.

The mechanism of MIA typically exploits the observation that ML models often behave differently on their training data (members) versus unseen data (non-members). This phenomenon is closely related to generalization gaps. A model will typically output prediction confidence scores with lower entropy (higher certainty) for data it has seen during training compared to data it has not.  
To execute this, adversaries often employ Shadow Modeling. The adversary creates multiple "shadow models" trained on data drawn from the same distribution as the target model (e.g., using public datasets or synthetic data). By observing the shadow models' behavior on known member and non-member inputs, the adversary trains a meta-classifier—a binary classifier that takes a target model's confidence vector as input and predicts "member" or "non-member". Research has shown that these attacks can achieve high accuracy even when the target model generalizes well, indicating that privacy leakage is not solely a function of overfitting.

#### **2.2.2 Model Inversion and Reconstruction Attacks**

While MIA reveals *presence* in a dataset, Model Inversion Attacks (MIAI) and Typical Instance Reconstruction (TIR) aim to recover the sensitive attributes or the raw data itself.
* **Attribute Inference**: An adversary with partial knowledge of a record (e.g., demographic data) accesses the model to infer a missing sensitive attribute (e.g., a genetic marker or financial status). The model's learned correlations serve to fill in the missing data points. If the model has learned that "individuals with X and Y characteristics always have condition Z" based on a training set that lacks diversity, it leaks the private attribute of those individuals.  
* **Deep Reconstruction (TIR)**: In Typical Instance Reconstruction attacks, adversaries utilize the model to generate near-accurate replicas of training samples. For example, in facial recognition systems, interrogating the model with optimization techniques can yield images that are visually indistinguishable from the original training photos. The adversary starts with a random noise image and iteratively perturbs it, following the gradient of the model's confidence score for a specific target class (e.g., "Person A"). Eventually, the noise converges into an image of Person A that the model recognizes with high confidence—effectively reconstructing the biometric features of the individual from the model weights alone.

#### **2.2.3 Gradient Leakage in Distributed Settings**

The rise of distributed learning, particularly in Federated Learning contexts where raw data remains local, introduced a new and potent attack vector: Gradient Inversion. The assumption was that sharing gradients (the direction and magnitude of model updates) was safe because it was not "data."

* **The Mechanism**: Gradients are derived directly from the input data via the chain rule of calculus. For a loss function $\mathcal{L}$ and weights $W$, the gradient is $\nabla W = \frac{\partial \mathcal{L}}{\partial W}$. This derivative contains a snapshot of the input data $x$.  
* **Deep Leakage from Gradients**: Research shows that an adversary (e.g., a malicious central server) can iteratively optimize a dummy input $\hat{x}$ and dummy label $\hat{y}$ such that their resulting gradient matches the received gradient $\nabla W$. By minimizing the Euclidean distance $\|\nabla W - \nabla W(\hat{x}, \hat{y})\|^2$, the adversary can recover the training data $x$. In image classification tasks, this technique has been shown to recover original training images with pixel-perfect accuracy, debunking the notion that gradients are privacy-preserving by default.

## **3\. The Failure of Traditional Data Handling**

Historically, privacy preservation was treated as a data-layer problem, addressed before the analytical process began. The prevailing methodology was "de-identification," "sanitization," or "anonymization." This approach posits that if one removes explicit identifiers, the remaining data is safe to use. However, the mathematical reality of high-dimensional data proves this assumption fundamentally flawed.

### **3.1 The Limits of De-identification**

The traditional approach involves scrubbing explicit identifiers (names, Social Security numbers) from a dataset. However, this relies on a binary distinction between "identifying" and "non-identifying" attributes that does not exist in reality. In high-dimensional datasets, a combination of seemingly innocuous attributes—Quasi-Identifiers (QIs) such as zip code, gender, and date of birth—can act as a unique composite key.13 A seminal study demonstrated that 87% of the U.S. population could be uniquely identified using only these three attributes.13 As datasets grow to include transaction times, location traces, or medical histories, the uniqueness of every individual record approaches 100%, rendering simple de-identification useless against linkage attacks where an adversary combines the anonymized dataset with an external, identified dataset (e.g., a voter registration list).

### **3.2 Syntactic Privacy Models and Their Collapse**

To counter linkage attacks, several syntactic privacy definitions were proposed. These models attempt to enforce statistical properties on the dataset to guarantee anonymity. While they provided a structured way to think about privacy, they ultimately proved insufficient for the complexity of modern machine learning data.

#### **3.2.1 k-Anonymity**

The concept of *k*\-anonymity requires that every record in a released dataset be indistinguishable from at least $k-1$ other records with respect to the Quasi-Identifiers.

* **Mechanism**: Data is generalized (e.g., replacing a specific age "24" with a range "20-30") or suppressed until groups of $k$ identical QI rows (equivalence classes) are formed.  
* **The Failure**: *k*\-anonymity provides aspirational rather than mathematical guarantees. It fails specifically against **Homogeneity Attacks**. If a specific $k$-group contains diverse individuals who all share the same sensitive attribute value (e.g., all 5 people in the "Age 20-30, Zip 90210" group have "Cancer"), the adversary learns the sensitive attribute simply by identifying the target's group, even without pinpointing the specific row. Furthermore, it does not protect against background knowledge attacks, where an adversary knows external facts that allow them to rule out members of the $k$-group.

#### **3.2.2 l-Diversity and t-Closeness**

Extensions were developed to patch *k*\-anonymity's flaws:

* **l-Diversity**: Requires that each equivalence class contains at least $l$ "well-represented" values for the sensitive attribute.13 This prevents the homogeneity attack but is susceptible to **Skewness Attacks**. If the overall distribution of a sensitive condition (e.g., HIV positive) is 1%, and an equivalence class has a 50% distribution, sensitive information is leaked probabilistically even if diversity exists. The adversary learns that the target is 50 times more likely to have the disease than the general population.  
* **t-Closeness**: Requires that the distribution of sensitive attribute values within each equivalence class is close to the distribution of the attribute in the overall dataset, within a threshold $t$.18 This attempts to mask the probabilistic leakage.

#### **3.2.3 The Curse of Dimensionality**

The fatal flaw for all these methods in the context of Machine Learning is the curse of dimensionality. These techniques rely on generalization (grouping similar records) and suppression (deleting distinct records). As the number of attributes (dimensions) increases, data points become increasingly sparse in the high-dimensional space. The "distance" between any two records grows, making it impossible to form groups of size $k$ without suppressing vast amounts of data or generalizing attributes to the point of uselessness (e.g., generalizing "Age" to "0-100"). 
For machine learning, which thrives on fine-grained, high-dimensional correlations, this is catastrophic. To achieve k-anonymity in a dataset useful for training a DNN (e.g., genomic data or pixel data), one would have to degrade the data quality so severely that the resulting model would have no predictive utility. The data becomes a shapeless blur. This failure of data-layer sanitization necessitates a paradigm shift toward Privacy-Preserving Machine Learning (PPML)—techniques that protect data during and after the computation, rather than just scrubbing it beforehand.

## **4\. Privacy-Preserving Machine Learning (PPML): The Technical Arsenal**

The field of PPML has evolved to address the dual challenges of data utility and rigorous privacy. It is critical to distinguish between techniques that protect **Data Confidentiality** (hiding input data during computation) and **Output Privacy** (preventing inference from the final model).

### **4.1 Technique I: Trusted Execution Environments (TEEs) / Confidential Computing**

Concept: Moving the trust boundary from software to hardware.  
Trusted Execution Environments (TEEs), often marketed under the umbrella of "Confidential Computing," utilize hardware-enforced isolation to create secure areas of memory, known as "enclaves." Code and data loaded into an enclave are protected from the operating system, the hypervisor, and even the system administrator with root privileges.

#### **4.1.1 Mechanism: Enclaves and Memory Encryption**

In a typical TEE implementation (e.g., Intel SGX, AMD SEV, or ARM TrustZone), the CPU creates a memory region where every cache line is encrypted instantly by a Memory Encryption Engine (MEE) located on the processor die.

* **Data in Use**: Unlike encryption at rest (storage) or in transit (network), TEEs protect data *in use*. The decryption keys exist only within the processor die and are inaccessible to software probing or cold boot attacks. 
* **Usage in ML**: An ML training workload can be placed inside an enclave. The raw data is encrypted by the data owner, sent to the untrusted cloud server, and decrypted only *inside* the CPU enclave for training. The model weights, gradients, and intermediate activations never appear in plaintext in the system RAM.

#### **4.1.2 Remote Attestation (RA)**

The cornerstone of TEE security is **Remote Attestation**. This is the cryptographic proof that allows a remote user to trust that the hardware is genuine and the code running inside is unmodified.

* **The Flow**:  
  1. The User challenges the Enclave.  
  2. The Enclave hardware generates a measurement (hash) of the initial code and data loaded into it.  
  3. This measurement is signed by a unique hardware key (provisioned by the manufacturer during fabrication) to produce a "Quote" or "Report". 
  4. The User verifies this Quote against the manufacturer's certificate service (e.g., Intel Attestation Service or via DCAP \- Data Center Attestation Primitives).  
  5. If verified, the user provisions the data decryption keys to the enclave via a secure channel (e.g., RA-TLS). This ensures that even if the cloud provider is malicious, they cannot access the keys because they cannot generate a valid attestation quote for malicious code.

#### **4.1.3 Issues and Limitations**

* **Confidentiality Only**: TEEs only protect the computation process. If the resulting model is released to the public, TEEs offer **zero protection** against Membership Inference Attacks or Model Inversion. The model inside the enclave can still overfit and memorize the data.  
* **Side-Channel Attacks**: TEEs share resources (caches, branch predictors) with the untrusted host OS. Adversaries can monitor access patterns (e.g., page faults, cache timing) to infer data processing characteristics inside the enclave.


**4.2 Technique II: Federated Learning (FL)**

Concept: Decentralized optimization; bring the code to the data.  
Federated Learning represents a fundamental architectural shift. Instead of centralizing data into a data lake, the model is sent to the devices (clients) where the data originates. The clients compute updates locally, and only the updates are sent back to the central server.

#### **4.2.1 The Federated Averaging (FedAvg) Algorithm**

The canonical algorithm for FL is **FedAvg**. It works in rounds of communication between the central server and $K$ clients.

Mathematical Formulation:  
Let $\mathcal{L}(w)$ be the global loss function we wish to minimize. The data is partitioned across $K$ clients, with client $k$ holding dataset $\mathcal{D}_k$ of size $n_k$. The total dataset size is $n = \sum n_k$.  
The objective is:

$$\min_{w} \mathcal{L}(w) = \sum_{k=1}^{K} \frac{n_k}{n} F_k(w)$$

where $F_k(w)$ is the local loss function on client $k$.  
**The Update Protocol**:

1. **Broadcast**: The server sends the current global model weights $w_t$ to selected clients.  
2. Local Training: Each client $k$ performs $E$ epochs of Stochastic Gradient Descent (SGD) on their local data $\mathcal{D}_k$ to obtain a new local weight vector $w_{t+1}^k$.

   $$w_{t+1}^k \leftarrow w_t - \eta \nabla F_k(w_t)$$  
3. Aggregation: Clients send $w_{t+1}^k$ (or the update $\Delta w = w_{t+1}^k - w_t$) back to the server. The server aggregates them using a weighted average:

   $$w_{t+1} \leftarrow \sum_{k=1}^{K} \frac{n_k}{n} w_{t+1}^k$$  
   

#### **4.2.2 Privacy Implications and Issues**

* **Data Locality**: The primary benefit is that raw data never leaves the device.  
* **No Inherent Privacy**: Sharing gradients $\Delta w$ is *not* synonymous with privacy. As discussed (Sec 2.2.3), a central server can reconstruct data from these updates (Gradient Leakage). FL is an **architectural** control, not a cryptographic or statistical privacy guarantee.


**4.3 Technique III: Differential Privacy (DP)**

Concept: Plausible deniability through mathematical noise.  
Differential Privacy (DP) is the only technique that rigorously addresses Output Privacy. It quantifies privacy loss and guarantees that the output of an algorithm does not depend significantly on any single individual's data, preventing the model from memorizing specific training examples.

#### **4.3.1 The Mathematical Definition ($\epsilon$-DP)**

A randomized algorithm $\mathcal{M}$ satisfies $(\epsilon, \delta)$-Differential Privacy if for all neighboring datasets $D$ and $D'$ (differing by exactly one record) and for all possible output sets $S \subseteq \text{Range}(\mathcal{M})$:

$$P(\mathcal{M}(D) \in S) \le e^{\epsilon} \cdot P(\mathcal{M}(D') \in S) + \delta$$

* **$\epsilon$ (Epsilon)**: The privacy budget. A smaller $\epsilon$ ensures that the model's behavior is nearly identical regardless of whether an individual's data is in the training set or not.  
* **$\delta$ (Delta)**: The probability that the privacy guarantee fails entirely.

#### **4.3.2 DP-SGD: Differentially Private Stochastic Gradient Descent**

Applying DP to Deep Learning involves modifying the gradient descent loop. The standard algorithm, **DP-SGD**, involves three key steps per iteration :

1. **Per-Sample Gradient Computation**: Compute the gradient $g_i = \nabla_{\theta} \mathcal{L}(\theta, x_i)$ for each individual example $x_i$.  
2. Gradient Clipping: To bound the sensitivity, clip the $L_2$ norm of each gradient to a threshold $C$.

   $$\bar{g}_i = g_i / \max\left(1, \frac{\|g_i\|_2}{C}\right)$$

   This prevents any single data point from having an unbounded influence on the model update. 
3. Noise Addition: Aggregate the clipped gradients and add Gaussian noise:

   $$\tilde{g} = \sum \bar{g}_i + \mathcal{N}(0, \sigma^2 C^2 \mathbb{I})$$  
4. **Descent**: Update the model weights using the noisy aggregate $\tilde{g}$.

**Role**: DP is the *only* tool listed here that prevents the final model from memorizing the input data.


**4.4 Technique IV: Secure Multi-Party Computation (SMPC)**

Concept: Computing on data you cannot see (Confidentiality).  
SMPC allows multiple parties to jointly compute a function over their inputs while keeping those inputs private. No party ever sees the raw data of another; they only see their own "share" of the data, which appears as random noise.

#### **4.4.1 Secret Sharing Schemes**

The most common primitive is **Shamir's Secret Sharing**. It relies on the geometric principle that $k$ points are required to define a polynomial of degree $k-1$.

* **Mechanism**: To share a secret $S$ among $n$ parties (with threshold $t$), the dealer generates a random polynomial $P(x)$ of degree $t-1$ such that $P(0) = S$.  
* Reconstruction: Using Lagrange Interpolation, any $t$ parties can reconstruct the polynomial and find $P(0)$.

  $$P(x) = \sum_{j=1}^{t} y_j \prod_{k \neq j} \frac{x - x_k}{x_j - x_k}$$

  Crucially, with fewer than $t$ shares, the secret is mathematically undetermined.

#### **4.4.2 Computing on Shares**

* **Secure Aggregation**: SMPC is ideal for aggregating gradients in Federated Learning. The server sums the updates without ever seeing an individual user's update.  
* **Role**: SMPC guarantees **Input Confidentiality** (the server cannot see the gradient). It does *not* guarantee **Output Privacy** (the sum of gradients might still leak information if not combined with DP).


**4.5 Technique V: Homomorphic Encryption (HE)**

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/ppml/fhe.png" class="img-fluid rounded z-depth-1" zoomable=true %}
    </div>
</div>

Concept: The Holy Grail of Cryptography (Confidentiality).  
Homomorphic Encryption allows computation directly on encrypted data. Ideally, $\text{Dec}(\text{Op}(\text{Enc}(m))) = \text{Op}(m)$. The server processes ciphertext and generates an encrypted result, never seeing the underlying data or the result.

#### **4.5.1 Fully Homomorphic Encryption (FHE) & CKKS**

For Machine Learning, the **CKKS** scheme is dominant because it supports approximate arithmetic on real/complex numbers (floating point).

* Mechanism: CKKS encodes vectors of numbers into polynomials in a ring $R_q = \mathbb{Z}_q[X]/(X^N+1)$. Encryption introduces a small "error" (noise) term $e$ for security.

  $$c = a \cdot s + m + e \pmod q$$  
* **Bootstrapping**: To allow infinite computations (Deep Learning), one must perform "Bootstrapping" to refresh the noise budget. This is computationally expensive.

#### **4.5.2 Role in Privacy**

Like SMPC, HE protects the **Confidentiality** of the input data and the computation process. It prevents the cloud server from stealing the data. However, **HE does not prevent the model from learning sensitive data.** If an FHE-trained model overfits, it still memorizes the data. The output is an encrypted model, but once decrypted for use, it is just as vulnerable to Membership Inference Attacks as a standard model.

## **5\. Comparative Analysis: Confidentiality vs. Privacy**

To provide a correct recommendation, we must dismantle the notion of a linear "Worst to Best" ranking. Instead, we must categorize these technologies by the specific privacy vector they address: **Process Confidentiality** (Can the builder see the data?) vs. **Output Privacy** (Can the user infer the data from the model?).

Table 1: The PPML Matrix

| Technique | Primary Function | Protects Against... | Fails Against... |
| :---- | :---- | :---- | :---- |
| **Data Anonymization** (k-anon) | Data De-identification | Direct lookup in DB | Linkage Attacks, High-Dim Re-ID |
| **Federated Learning (FL)** | Data Minimization | Centralized Data Breach | Gradient Leakage, Model Memorization |
| **FHE / SMPC / TEE** | **Confidentiality** (Process) | Server/Insider Attacks | **Model Memorization**, Inference Attacks |
| **Differential Privacy (DP)** | **Privacy** (Output) | **Model Memorization**, Inference Attacks | Model Accuracy Degradation |

### **5.1 The Misconception of Encryption (FHE/SMPC) as "Privacy"**

It is a critical error to assume that FHE or SMPC provides "privacy" for the subject of the data.

* **What they do**: They ensure that the *entity performing the training* (e.g., the cloud provider or the aggregator) cannot access the raw data or the individual gradients. They provide **Input Confidentiality**.  
* **What they fail to do**: They do *not* change *what* the model learns. If you train a model using FHE on a dataset of sensitive medical records, the mathematics of the training are identical to training on plaintext. If that training process leads to overfitting, the resulting model (once decrypted) will still contain memorized patient data. An adversary querying the final model can still perform a Membership Inference Attack.

### **5.2 The Unique Role of Differential Privacy (DP)**

DP is the only technique that modifies the *learning process itself* to prevent the model from learning too much about any single individual.

* **What it does**: By adding noise (e.g., via DP-SGD), it mathematically smooths the model's decision boundaries. It ensures **Output Privacy**.  
* **What it fails to do**: On its own, DP does not hide the raw data from the server training the model (unless Local DP is used, which often destroys utility).

### **5.3 The Hierarchy of Solutions (Revised)**

Instead of "Worst to Best," we progress from **Vulnerable** to **Robust Hybrid Architectures**.

#### **Level 1: Vulnerable (Standard ML)**

* **Technique**: Centralized training on raw data.  
* **Risk**: Total exposure. Data breaches, insider threats, and model memorization are all possible.

#### **Level 2: Architectural Minimization**

* **Technique**: **Federated Learning (FL)**.  
* **Benefit**: Data stays on device.  
* **Risk**: The central server can still reconstruct data from gradients; the final model still memorizes data.

#### **Level 3: Input Confidentiality (The "Secure Pipe")**

* **Technique**: **FL \+ SMPC** (Secure Aggregation) OR **FHE**.  
* **Benefit**: The server *cannot* see individual updates or raw data.  
* **Risk**: The *final global model* is still a perfect summary of the training data. If released, it leaks privacy.

#### **Level 4: Output Privacy (The "Secure Result")**

* **Technique**: **Centralized Training \+ DP-SGD**.  
* **Benefit**: The final model is mathematically guaranteed not to memorize individuals.  
* **Risk**: The entity training the model must be trusted with the raw data.

#### **Level 5: The "Gold Standard" (Hybrid)**

* **Technique**: **FL \+ SMPC \+ DP**.  
* **Benefit**:  
  1. **FL**: Data never leaves the device.  
  2. **SMPC**: The server never sees individual gradients (only the noisy sum).  
  3. DP: The final model (and the sum seen by the server) contains noise that prevents reconstruction of any single user's data.  
     This combination protects data at rest, in transit, during computation, and after release.

## **6\. Conclusion**

The landscape of Privacy-Preserving Machine Learning is often clouded by the conflation of "secrecy" and "privacy." Technologies like **Homomorphic Encryption (FHE)** and **Secure Multi-Party Computation (SMPC)** are powerful cryptographic tools for **Confidentiality**—they allow us to compute on data we cannot see. However, they are mathematically invisible to the learning algorithm; they do not prevent the model from memorizing the secrets they protect.

True privacy—the guarantee that a model reveals nothing about its subjects—requires **Differential Privacy (DP)**. DP is the only mechanism that inoculates the model against inference attacks. Consequently, the optimal strategy is not to choose *between* these technologies, but to layer them. FHE and SMPC blind the server; DP blinds the model.

## **7\. Strategic Recommendations for Implementation**

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/ppml/Hybrid.png" class="img-fluid rounded z-depth-1" zoomable=true %}
    </div>
</div>

Based on the distinction between input confidentiality and output privacy, we recommend the following implementation strategies:

### **7.1 Scenario A: Single Party with Sensitive Data (Internal Use)**

* **Goal**: Protect against model memorization (e.g., preventing a released model from leaking internal employee data).  
* **Recommendation**: **DP-SGD**.  
  * Use techniques like **k-anonymity** as a preprocessing step to reduce the sensitivity of outliers, but rely on **Differential Privacy** (Gradient Clipping + Noise) during training to ensure the model weights do not encode user-specific rows.

### **7.2 Scenario B: Multi-Party Collaborative Training (e.g., Hospital Consortium)**

* **Goal**: Train a joint model without sharing patient records between hospitals or with the central server.  
* **Recommendation**: **Federated Learning (FL) + Secure Aggregation (SMPC) + Differential Privacy (DP)**.  
  * **FL**: Keeps patient records at each hospital.  
  * **SMPC**: Ensures the central server only sees the *sum* of the updates, preventing it from reconstructing data from a specific hospital.  
  * **DP**: Adds noise to the updates. This ensures that even the *aggregated* model does not memorize a rare patient case from one hospital.

### **7.3 Scenario C: Outsourced Computation (Untrusted Cloud)**

* **Goal**: Use a third-party cloud to train a model on your data without the cloud provider seeing the data.  
* **Recommendation**: **FHE (Fully Homomorphic Encryption)** or **TEEs (Confidential Computing)**.  
  * **Why**: These provide **Confidentiality**. The cloud processes encrypted data (FHE) or hardware-protected data (TEE).  
  * **Critical Note**: If you plan to *release* this model to the public later, you **must also apply DP** during the training. FHE only protects the data from the *cloud provider*, not from the *end user* who queries the model.

By understanding that **FHE/SMPC protects the process** and **DP protects the result**, organizations can architect solutions that are truly robust against the full spectrum of modern privacy attacks.

#### **References**

1. Membership Inference Attacks Against Machine Learning Models - Cornell: Computer Science, accessed December 29, 2025, [https://www.cs.cornell.edu/~shmat/shmat_oak17.pdf](https://www.cs.cornell.edu/~shmat/shmat_oak17.pdf)  
2. Differential Privacy Series Part 1 | DP-SGD Algorithm Explained | by PyTorch - Medium, accessed December 29, 2025, [https://medium.com/pytorch/differential-privacy-series-part-1-dp-sgd-algorithm-explained-12512c3959a3](https://medium.com/pytorch/differential-privacy-series-part-1-dp-sgd-algorithm-explained-12512c3959a3)  
3. Bridging the gap in differentially private model training - Google Research, accessed December 29, 2025, [https://research.google/blog/bridging-the-gap-in-differentially-private-model-training/](https://research.google/blog/bridging-the-gap-in-differentially-private-model-training/)  
4. (PDF) A Survey of Privacy Attacks in Machine Learning - ResearchGate, accessed December 29, 2025, [https://www.researchgate.net/publication/373958404_A_Survey_of_Privacy_Attacks_in_Machine_Learning](https://www.researchgate.net/publication/373958404_A_Survey_of_Privacy_Attacks_in_Machine_Learning)   
5. Model Inversion Attacks: A Survey of Approaches and Countermeasures - arXiv, accessed December 29, 2025, [https://arxiv.org/html/2411.10023v1](https://arxiv.org/html/2411.10023v1)  
6. Secure Aggregation in Federated Learning using Multiparty Homomorphic Encryption - arXiv, accessed December 29, 2025, [https://arxiv.org/pdf/2503.00581](https://arxiv.org/pdf/2503.00581)  
7. l-Diversity: Privacy Beyond k-Anonymity - Cornell: Computer Science, accessed December 29, 2025, [https://www.cs.cornell.edu/johannes/papers/2005/publishing-icde-final.pdf](https://www.cs.cornell.edu/johannes/papers/2005/publishing-icde-final.pdf)   
8. Protecting Sensitive Data and AI Models with Confidential Computing - NVIDIA Developer, accessed December 29, 2025, [https://developer.nvidia.com/blog/protecting-sensitive-data-and-ai-models-with-confidential-computing/](https://developer.nvidia.com/blog/protecting-sensitive-data-and-ai-models-with-confidential-computing/)  
9. Federated Learning Algorithm Averaging Methods FedAvgM, FedProx, FedAvg - Computing Notes, accessed December 29, 2025, [https://computingnotes.com/federated-learning-algorithm-averaging-methods-fedavgm-fedprox-fedavg/](https://computingnotes.com/federated-learning-algorithm-averaging-methods-fedavgm-fedprox-fedavg/)  
10. Differential Privacy | Harvard University Privacy Tools Project, accessed December 29, 2025, [https://privacytools.seas.harvard.edu/differential-privacy](https://privacytools.seas.harvard.edu/differential-privacy)  
11. Analyzing and Optimizing Perturbation of DP-SGD Geometrically - arXiv, accessed December 29, 2025, [https://arxiv.org/html/2504.05618v1](https://arxiv.org/html/2504.05618v1)  
12. CKKS Scheme - The Beginner's Textbook for Fully Homomorphic Encryption, accessed December 29, 2025, [https://fhetextbook.github.io/CKKSScheme.html](https://fhetextbook.github.io/CKKSScheme.html)  