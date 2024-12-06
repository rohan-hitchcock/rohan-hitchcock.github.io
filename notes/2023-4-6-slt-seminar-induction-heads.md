---
title: Talk on In-Context Learning and Induction Heads
layout: page
katex: true
---
In this talk we discuss the paper Olsson et al (2022) In-context Learning and Induction Heads. The paper discusses *induction heads*, which are a proposed emergent structure in transformers which implements a particular algorithm. 

The hypothesis of this paper is that induction heads constitute the mechanism for the the majority of in-context learning in transformers. It presents the following six arguments towards this claim:
1. During training, three events appear to coincide: there is a sudden drop in training loss, the model gains the ability to do in-context learning, and induction heads form.
2. When the model architecture is changed to facilitate forming induction heads the in-context learning ability of the model increases or is gained earlier in training.
3. When induction heads are removed during testing the model loses the ability to do in-context learning.
4. In some circumstances, such as when doing translation, induction heads appear to perform a more sophisticated "fuzzy" version of their algorithm.
5. For small models the authors reverse-engineer induction heads and show how they contribute to in-context learning.
6. "Many behaviours and data related to both induction heads and in-context learning are smoothly continuous from small to large models."

We will focus on 1. 

## Model architectures

All models are GPT-style transformers using multi-head attention.
- **Small models:** Have between 1 and 6 layers and 12 attention heads per layer. Each has two versions: one with only attention layers and one with both attention layers and fully connected layers. I'll show figures for the attention-only models.
- **Large models:** Similar but bigger with fully-connected layers. Six models ranging from 4 layers with 13M parameters to 40 layers with 13B parameters. 


## Detecting induction heads

**Definition:** An *induction head* is an attention head which exhibits a type of 'completion algorithm' behaviour: for any tokens A and B, given a sequence of the form ...AB...A it influences the prediction of the next token to be B. That is, an induction head looks for the previous occurrence of the current token and 'predicts' that the next token will be the one that followed the current token last time. 

- One might also consider 'fuzzy' versions of this. For any tokens $$A$$, $$B$$, $$A'$$, $$B$$' where $$A \approx A'$$ and $$B\approx B'$$ in some sense, a 'fuzzy' induction head would predict $$B'$$ when given the sequence $$\ldots AB\ldots A'$$. 

The formation of induction heads is primarily measured using a proxy called the *prefix matching score*. 

**Definition:** Given an attention head, its *prefix matching score* is computed as follows. A sequence of 25 random tokens (excluding the most and least common tokens) is generated and repeated four times and a start-of-sequence token is added. We record the attention scores of each token. The *prefix matching score* is the average attention assigned to the token preceding the current token in the earlier repeats.

## Measuring in-context learning

The idea of in-context learning is that the model should get better at next-token prediction further into the sequence. Hence:

**Definition:** The *in-context learning score* of a model is the average loss of the 500th token minus the average loss of the 50th token. Test sequences have 512 tokens.

Note that a more negative in-context learning score should demonstrate a higher level of in-context learning. 

The authors claim that the arbitrary choices in this definition don't change things much. I think its not too hard to come up with something better. For example, one could average the loss of tokens near the start and end and look at that difference. Or, look at the gradient of the line of best fit to this index vs loss graph. 

## Argument 1: the phase change

The most interesting part of this paper, in my opinion, is the "phase change" occurring early in training. This shows up in the training loss, the in-context learning score and the prefix matching score. (Look at the graphs).

Note that:
- Models are small, attention only.
- The yellow window is in the exact same place on each graph.
- On the prefix-matching graph each curve is an attention head, so the ones which 'jump up' are the hypothesised induction heads and the other heads are doing something else. 
- We shouldn't expect induction heads to form in 1 layer attention-only models. A model consisting of a single attention head layer can only use the value of each token to compute the attention score, but induction heads need to perform an *algorithm* which looks at a token matching the current token, then looks at the token immediately following it. Eg in AB...A it should attend to B. The only way for this to occur is for a previous layer to encode the fact that B follows an A token in the updated embedding of B. 
- This event is also observed in other quantities such as the derivative of loss with respect to token index, and the trajectory of the principal components of a quantity called the "per token loss".

Similar observations are also made for small models with fully-connected layers and for large models, but things are a bit messier. 

## Argument 2: facilitating induction heads

They add a mechanism which allows the model to learn how to mix information between 'key' vectors in the same induction head:

$$
\tilde{k}_t^h = \sigma(\alpha^h) k_t^h + (1 - \sigma(\alpha^h)) k_{t-1}
$$

where $$\alpha ^h$$ is a trainable parameter. When this is introduced, the phase change above is observed earlier in training for >2 layer models and is observed in 1 layer models. 

This experiment is done only for small models.

## Argument 3: removing induction heads

They basically delete various attention heads at test time and try to attribute the amount of in-context learning they are responsible for. From their figures it seems that these coincide with the induction heads, identified in argument 1. 

I wish they had given more detail about these experiments. 

## Argument 4: induction heads doing other things

In their large models, they look at the behaviour of the heads identified (using prefix matching score) as induction heads. They qualitatively observe that in some tasks, some of these heads do something which is like a fuzzy version of copying. 

For example, one specific induction head, when presented with the same text in three different languages, exhibits a similar attention pattern: attending to the token following the translated version of the current token.

A different specific induction head was able to match an constructed pattern in a similar way. 

## Argument 5: reverse engineering induction heads

Actually, most of this work occurs in the previous paper  Elhage et al (2021) "A mathematical framework for transformer circuits". It describes attention in-terms of tensor products, and provides examples of how an induction head could be implemented. This could be a good thing to look at in a future talk.

However, induction heads can be implemented in many different ways, and different implementations have been observed.

## Unexplained observations (by the authors):

- The in-context learning score is approximately constant and **the same for all models** after the phase change. This includes all models from the toy 2-layer attention only model to the model with 13B parameters. Note that the large models are still much better in-terms of absolute performance. This statement is just saying that the ability to learn from from context is the same for all models. 
- Consider the derivative of the training loss as a function of training 'time', aka *training speed*. Of the large models (13M - 13B), the smaller learn faster before the phase transition, and the larger learn faster after the phase transition. 
