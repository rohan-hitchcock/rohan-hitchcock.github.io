---
title: Metauni AI Safety seminar 19 October 2023
layout: page
katex: true
date: 2023-10-19
---

*Notes for a talk I gave the the [metauni AI Safety reading group](https://metauni.org/ai-safety/)*

---

In this seminar we look at the paper "What does it take to catch a Chinchilla?
Verifying rules on Large-Scale Neural Network Training via Compute Monitoring"
by Yonadav Shavit. 

This paper proposes a solution which addresses the following situation:
- There are agreed rules which limit large scale development of AI systems.
- There are AI race dynamics (eg between companies or countries). All actors would 
prefer to follow the rules and develop AI systems safely (or entirely forgo developing
dangerous models), but they are under pressure 
to sacrifice safety to deploy high-capability systems faster as they have no way 
of verifying whether their competitors are doing the same.  

This paper should be read as one which organises and clearly states a number of 
technical problems which, if solved, would constitute a mechanism to enforce rules 
on training frontier models. 

Some limitations of the proposed solution:
- Only focussed on training runs done in large, specialised, known data centers. 
- Chips produced before such a framework comes into place will escape monitoring 
- Doesn't apply to small models
- Once a model is created it doesn't restrict its proliferation

## Problem statement

We consider the following setting. We have two parties: the *Verifier* and the 
*Prover*. The Verifier's goal is to verify (with high probability) that the 
Prover is complying with AI training rules, and the Prover's wants to convince 
the verifier that it is. The Verifier can request the Prover take actions, such as 
disclosing information about training runs.
We assume that the Prover is a *covert adversary* (Aumann and Lindell 2010), 
meaning that they want to violate the training rules but only if they can still 
appear compliant to the Verifier. 

We further impose the constraint that the Prover does not want to disclose confidential 
information about how their (legitimate) models are being trained, and the Verifier 
accepts this. In other words, the Prover should not be required to disclose 
information which would allow the Verifier to reproduce their models, or otherwise
gain an advantage. Ideally the only information the Verifier can gain is whether
or not the Prover has complied with the rules.

The paper highlights two real-world Prover-Verifier relationships:
- **Domestic Oversight:** A government enforcing regulations on companies within 
its borders.
- **International Oversight:** Internationally agreed rules where the Prover is 
a government or large company and the Verifier is another government or international 
body. For example the [START treaties](https://en.wikipedia.org/wiki/START_I) between 
the US and the USSR/Russia, which allow each party to inspect one another's nuclear 
weapons facilities.

The central problem is:
> What what minimal set of verifiable actions can the Verifier require the Prover 
> to take that would enable the Verifier to detect, with high probability, whether 
> the Prover violated any training rules?

### Types of rules

The paper lists several examples of properties of training runs which could be 
constrained.
- **Total training compute:** *cf.* scaling laws.
- **Properties of the training data** such as whether the dataset contains code 
for cybersecurity exploits.
- **Properties of the hyperparameters** such as the amount of reinforcement 
learning used.
- **Benchmark performance:** such as on benchmarks designed to determine a 
model's capabilities. 
- **Combinations of other rules** such as "if the model was trained 
using RL for more than X GPU days then it should not exceed performance level Y 
on benchmark Z".

However, the paper notes this as an open problem: "Current 'scaling law' 
extrapolations are not yet able to reliably predict ML models’ downstream 
capabilities, so finding principled methods for deciding on rule-thresholds 
that achieve desired policy outcomes is an important area for future work.". 
*Cf.* devinterp.

## Monitoring protocol

The entire monitoring protocol can be broken down into three parts:


### 1. On-chip logging
During a large training run, the neural network's weights 
are split across a large number of chips. We call this a *weight shard*. Intermittently 
(and randomly) over training the weight shard $$$$w$$$$ on each chip is saved. 
The Prover retains a copy of $$$$w$$$$, and a hashed version $$$$h = H(w)$$$$ is 
retained for the Verifier This could be stored securely on the chip so 
the Verifier can retrieve it by physical inspection, or having the chip 
sign the hash with a private key and sending it to the verifier. A 
physical inspection would also mean the Verifier could check whether the 
chip had been tampered with. 

### 2. Training run verification and proof of training transcript
The Verifier is allowed to choose any chip and weight shard to inspect. Once 
a weight shard $$$$w$$$$ is chosen, it is the responsibility of the 
Prover to convince the Verifier that (with high probability) that the run which 
generated this weight shard 
did not violate any of the rules. Precisely how this is done is an open problem;
we consider a candidate protocol below. 

The paper derives lower bounds (in terms of the number of chips a Prover owns) 
in the sampling rate of chips to ensure a single rule violating run is caught 
with probability $$$$p$$$$. See paper, but the number of samples does not appear unreasonable.

For every training run the Prover retains a *training transcript* $$$$T$$$$, which contains 
all the information to deterministically reproduce the training run (including 
but not limited to: model architecture, dataset (and ordering), random seeds, 
hyperparameters). If we assume, given $$$$T$$$$, that $$$$w$$$$ may be deterministically 
reproduced *and* that it is very difficult for the Prover to find a second 
training transcript $$$$T' \neq T$$$$ which also produces $$$$w$$$$. Given these assumptions 
the Prover could reveal the training transcript $$$$T$$$$ to the Verifier and retrain the 
model to reproduce the shard $$$$w$$$$ (which the Verifier observes). This has some 
obvious problems:
1. The Prover would likely be unwilling to agree to reveal $$$$T$$$$, as this reveals 
confidential information.
2. The verification procedure is extremely expensive.
3. The assumption that the training process is deterministic is not realistic (eg
due to noise in computer hardware).
4. It is not clear whether it is genuinely difficult to find fraudulent training 
transcripts.

We first consider a better verification procedure and then address point 1.

#### Proof of learning and proof of training transcript
While the *proof of training transcript* (PoTT) problem (proving that a given transcript 
results in a weight shard $$w$$) has not yet been studied, a related problem called 
*proof of learning* (Jia et al. 2021) (PoL) has. In this problem, the Prover wants to prove that 
they 'own' a given, trained model (eg. to refute intellectual property claims). 
Let $$W$$ be the final weights of the model (which the Prover wants to prove ownership
of). If they have legitimately trained the model, the Prover retains the training 
transcript $$T$$ and previously agreed model snapshots $$W_0, W_1, W_2, \ldots$$. A
Verifier first checks that the initialisation $$W_0$$ is likely random, and then 
uses some procedure to choose pairs of weight snapshots $$(W_i, W_{i+k}), \ldots, (W_j, W_{j+k})$$
which are $$k$$ steps apart. These, along with the training transcript, are revealed to the Verifier
and these sections of training are rerun. The verifier confirms that running training
from $$W_i$$ according to $$T$$ for $$k$$ steps (approximately) results in $$W_{i+k}$$. 
Various procedures for the verifier are proposed in the literature. "Heuristic-based"
method are proposed but it remains the subject of ongoing work.

PoTT differs from PoL in the following ways:
1. Regardless of whether or not the Prover is behaving legitimately, they know 
the correct training transcript $$T$$ to produce the requested weight shard $$w$$.
2. An adversarial Prover may be willing to spend more than the original training 
compute to construct a spoofed training transcript $$T'$$ which complies with the rules.
3. The PoL problem considers the situation where the entire weight vector is known 
to the Verifier, rather than just a weight shard. 

The paper suggests that the PoTT problem seems strictly harder than PoL, however 
it is conceivable that a PoL protocol may be used for PoTT. The paper states the 
following definition by which to evaluate a PoTT protocol.

**Definition:** A *PoTT protocol* $$(R, V)$$ consists of a Prover's reporting 
protocol $$R$$ and a Verifiers verification protocol $$V$$. Given a target weight vector 
$$W$$ and training protocol $$T$$, if $$W$$ was legitimately produced from $$T$$ then 
the Verifier should accept with high probability:

$$
\mathbf{P}(V(T, W) = \text{accept}) > 1 - \delta
$$

for some small $$\delta$$. A *spoofed training transcript* $$T'$$ for a weight vector 
$$W$$ is a training transcript which is sufficiently different from the True training 
transcript $$T$$. A PoTT protocol us *$$J$$-efficient and $$F$$-hard* if $$V$$ is
$$\mathcal{O}(J)$$ and there does not exist a $$\mathcal{O}(F)$$ spoof-generation 
algorithm $$A$$ such that 

$$
\mathbf{P}(V(A(T), W) = \text{accept}) > 1 - \delta '
$$

for some (fixed) small $$\delta'$$. 

**Some notes:**
- While PoTT is likely harder than PoL, PoTT has the advantage that the Verifier 
does not have to reveal their verification strategies to the Prover (and can use 
multiple). This means it is more difficult to come up with an adversarial 
spoofing algorithm which attacks the *specific* verification strategy $$V$$.
- Require the Prover to *pre-commit* portions of its training transcript and 
log a (hashed) version with the weight snapshot. This constrains the scope of 
an adversarial Prover to come up with spoofed training transcripts.
- On point 3 (weights vs weight-shards), the paper presents the view that it is 
likely to be just as hard to find a training transcript which is a spoof for a 
weight shard as it is for the entire weight vector. It however is an open problem. 

#### Keeping the training transcript confidential

The paper proposes the following method. Upon requesting to inspect a shard:
1. The Prover provides a *hashed* training transcript to the Verifier. 
2. The Prover and Verifier agree on a *trusted environment* to execute the verification 
procedure. That is, one where the Prover trusts the Verifier cannot observe sensitive information 
and the Verifier trusts will legitimately perform any tests it asks.
3. Within the trusted environment, the Prover supplies the training transcript and 
the Verifier asks for tests to be performed. That is:
    - Check whether training transcript and weight shard both produce the hash 
    values provided to the Verifier.
    - Perform the Verifier's protocol $$V$$ to check whether the weight shard 
    was legitimately produced from the training transcript.
    - Run any other tests on $$T$$ which check whether it complies with the rules.

### 3. Supply chain 
Basically we need to know where all the chips are. I don't know very much about this, 
but the paper seems to suggest that a lot of this infrastructure is already used 
for other purposes eg. when the US exports military technology to its allies. 

## Bibliography
- Shavit (2023) “What does it take to catch a Chinchilla? Verifying Rules 
on Large-Scale Neural Network Training via Compute Monitoring.” http://arxiv.org/abs/2303.11341
- Aumann and Lindell (2010) “Security Against Covert Adversaries: 
Efficient Protocols for Realistic Adversaries,” Journal of Cryptology, 23.2 pp. 281–343
- Jia et al. (2021) “Proof-of-Learning: Definitions and Practice,” in 2021 IEEE Symposium on Security and Privacy 
