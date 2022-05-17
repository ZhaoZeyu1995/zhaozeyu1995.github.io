---
title:  "CTC Prefix Beam Search Decoding Algorithm with Language Model"
mathjax: true
---

This short article introduces the basis of CTC Prefix Beam Search Decoding Algorithm, which was a little bit difficult for me in the past. 
I found a very useful [article on Medium](https://medium.com/corti-ai/ctc-networks-and-language-models-prefix-beam-search-explained-c11d1ee23306), 
and here is the [github repo](https://github.com/corticph/prefix-beam-search) for it. 

In this article, we mainly focus on the file `prefix_beam_search.py`, 
which contains the code all we need to understand. 
Before reading this article, you are assumed to know the basic concept of CTC and language model. 
Basically, CTC is short for Connectionist Temporal Classification, 
and a loss function defined between two different sequences but without alignments. 
For example, in the field of Automatic Speech Recognition (ASR), 
the basic method is to define a loss (or objective) function given input speech features and the corresponding transcriptions. 
However, there is no readily available alignments in transcriptions. 
Thus, the essential idea of CTC is to calculate the term $p(l|x)$ by marginalising all the possible alignments between the input $x$ and the label $l$. 
The language model can output a probability for any input word sequences. 

Below is the copy of the code.

```python
from collections import defaultdict, Counter
from string import ascii_lowercase
import re
import numpy as np

def prefix_beam_search(ctc, lm=None, k=25, alpha=0.30, beta=5, prune=0.001):
	"""
	Performs prefix beam search on the output of a CTC network.

	Args:
		ctc (np.ndarray): The CTC output. Should be a 2D array (timesteps x alphabet_size)
		lm (func): Language model function. Should take as input a string and output a probability.
		k (int): The beam width. Will keep the 'k' most likely candidates at each timestep.
		alpha (float): The language model weight. Should usually be between 0 and 1.
		beta (float): The language model compensation term. The higher the 'alpha', the higher the 'beta'.
		prune (float): Only extend prefixes with chars with an emission probability higher than 'prune'.

	Retruns:
		string: The decoded CTC output.
	"""

	lm = (lambda l: 1) if lm is None else lm # if no LM is provided, just set to function returning 1
	W = lambda l: re.findall(r'\w+[\s|>]', l)
	alphabet = list(ascii_lowercase) + [' ', '>', '%']
	F = ctc.shape[1]
	ctc = np.vstack((np.zeros(F), ctc)) # just add an imaginative zero'th step (will make indexing more intuitive)
	T = ctc.shape[0]

	# STEP 1: Initiliazation
	O = ''
	Pb, Pnb = defaultdict(Counter), defaultdict(Counter)
	Pb[0][O] = 1
	Pnb[0][O] = 0
	A_prev = [O]
	# END: STEP 1

	# STEP 2: Iterations and pruning
	for t in range(1, T):
		pruned_alphabet = [alphabet[i] for i in np.where(ctc[t] > prune)[0]]
		for l in A_prev:
			
			if len(l) > 0 and l[-1] == '>':
				Pb[t][l] = Pb[t - 1][l]
				Pnb[t][l] = Pnb[t - 1][l]
				continue  

			for c in pruned_alphabet:
				c_ix = alphabet.index(c)
				# END: STEP 2
				
				# STEP 3: “Extending” with a blank
				if c == '%':
					Pb[t][l] += ctc[t][-1] * (Pb[t - 1][l] + Pnb[t - 1][l])
				# END: STEP 3
				
				# STEP 4: Extending with the end character
				else:
					l_plus = l + c
					if len(l) > 0 and c == l[-1]:
						Pnb[t][l_plus] += ctc[t][c_ix] * Pb[t - 1][l]
						Pnb[t][l] += ctc[t][c_ix] * Pnb[t - 1][l]
				# END: STEP 4

					# STEP 5: Extending with any other non-blank character and LM constraints
					elif len(l.replace(' ', '')) > 0 and c in (' ', '>'):
						lm_prob = lm(l_plus.strip(' >')) ** alpha
						Pnb[t][l_plus] += lm_prob * ctc[t][c_ix] * (Pb[t - 1][l] + Pnb[t - 1][l])
					else:
						Pnb[t][l_plus] += ctc[t][c_ix] * (Pb[t - 1][l] + Pnb[t - 1][l])
					# END: STEP 5

					# STEP 6: Make use of discarded prefixes
					if l_plus not in A_prev:
						Pb[t][l_plus] += ctc[t][-1] * (Pb[t - 1][l_plus] + Pnb[t - 1][l_plus])
						Pnb[t][l_plus] += ctc[t][c_ix] * Pnb[t - 1][l_plus]
					# END: STEP 6

		# STEP 7: Select most probable prefixes
		A_next = Pb[t] + Pnb[t]
		sorter = lambda l: A_next[l] * (len(W(l)) + 1) ** beta
		A_prev = sorted(A_next, key=sorter, reverse=True)[:k]
		# END: STEP 7

	return A_prev[0].strip('>')
```

First of all, let's have a look at the input parameters of this function. 
There has already some comments in this code file, so here is some of my personal understanding. 

1.  `ctc` is the output of neural networks, with shape of `(time_steps, num_tokens)`
2.  `lm` is the language model 
3.  `k` , the beam width. 
4.  `alpha`, the hyperparameter for the language model. 
5.  `beta`, compensation hyperparameter, I am that sure for this currently (I am still learning...)
6.  `prune`, actually, this is a threshold for the first input argument `ctc`, and, basically, it means the at each and every time step, the token with a lower probability than this threshold will be ignored or pruned. 

From line 22 to 27, this is the preparation part. In line 22, this will generate a LM function if there is no input LM. 
In line 23, this is a regular expression whose function is still unknown to me. 
The alphabet is defined in line 24, where `list(ascii_lowercase)` would return all the lower case letters from a to z. 
Besides, the alphabet also contains `[' ', '>', '%']` which denote <space>, <end of sentence>, and <blank> label in CTC, respectively. 
From line 25 to 27, we know `F` represents the number of the output tokens and `T` for the number of time steps. In line 26, the comment at the end of that line shows its meaning. 

## Initialisation Step 

From line 30 to 34, firstly, `O` is defined to be a empty string, which is logical because we should start from a empty string. 
After that, in line 31, two important variables, Pb and Pnb, are defined. Both of them are `defaultdict` in python. 
They can be used as the form like `Pb[t][l]`, where `t` represent the time step and `l` represent the prefix. 
For `Pb`, it represents the probability of a given prefix `l` at time `t` ending with a <blank> token and `Pnb`, of course, denotes the probability ends with a non-blank token. 
As there is initially no token in the prefix string `O`, so we should simply initialise `Pb[0][O]=1` and `Pnb[0][O]=0` as we read from line 32 to 33. 
If you cannot understand this, don't worry because you will definitely do after reading the following. `A_prev=[O]` is the initial prefix set. 

## Iterations and pruning 

Next, we can see that there is a for-loop from 1 to `T`. 
First of all, in line 39, the `prune` parameter is applied as a threshold, so we only need to consider the tokens with high enough probability. 
After that, we need to process the existing prefixes in `A_prev` in different ways accordingly. 
From line 40, we enter the next internal for-loop to process all prefixes in `A_prev`. 
From line 42 to 45, we deal with the prefixes ending with `'>'` which is the <end of sentence> token by copying its probability from time step `t-1` to `t`. 
Actually, a prefix that ends with a <end of sentence> token is a complete decoding result already. 
From line 47 on, we enter another for-loop with respect to different tokens in the `pruned_alphabet`, and from line 48, we can know that `c_ix` represents the index of the token `c`. 
For each token in `pruned_alphabet`, first of all, if it is a `'%'`, <blank> token, the prefix `l` will not be changed. 
However, after adding the <blank> token to the prefix `l`, it becomes a prefix which ends with <blank> token, so we have 

```python
Pb[t][l] += ctc[t][-1] * (Pb[t-1][l] + Pnb[t-1][l])
```

where the original value of `Pb[t][l]` should be zero and `ctc[t][-1]` denotes the probability of <blank> token. 
Note that, in this code, the index of <blank> token is `-1` but in ESPnet, it is usually `0` instead. 
From line 57 to 61, if the token being processed is not <blank>, we will extend the prefix `l`with token `c` to get `l_plus`. 
Besides, we need to update the probability regarding the new prefix `l_plus`, but before that we need to be careful with different situations. 

`if len(l) > 0 and c == l[-1]`, in this case, the `l_plus` can be a genuin new prefix only if prefix `l` ends with a <blank> token. 
That's why we multiply `ctc[t][c_ix]` and `Pb[t-1][l]` in line 60. 
On the contrary, if `l` ends with a non-blank token which is the same as `c`, these repeated same tokens will collapse into one single token. 
Thus, we have line 61. 
Till now, we have considered the case in which `c` is the same as the last token in `l` and now we need to consider if the new token is <space> or <end of sentence> to apply our language model. 
In line 65, we can see that 

```python
elif len(l.replace(' ', '')) > 0 and c in (' ', '>'):
```

which is just the case that we would like to consider currently, where the former part is for making sure that this prefix `l` is not full of <space> or even empty. 
When `c` is <space> or <end of sentece>, it means that we have outputted a completed new word. 
So it is time to apply our language model now. 
From line 66, we can see that the hyperparameter `alpha` is used as an index number and we get the language model probability `lm_prob`. 
After that, we update the `Pnb[t][l_plus] += lm_prob * ctc[t][c_ix] * (Pb[t-1][l] + Pnb[t-1][l])`. 
Finally, if `c` is different from the last token of  `l` and it does not indicate an end of a new word, 
we would just update the probability concerning both `Pnb` and `Pb` at the same time. 
Here, you may be curious that why we use `+=` to update the probability but not `=` directly. 
That's because to get a prefix `l` we may have different decoding paths, 
which is the essential concept of CTC, calculating the loss without the alignment by marginalising it. 
At the same time, you would also understand why we initialise `Pnb[0][O]` as zero and `Pb[0][O]` as one. 
That's because an empty prefix behaves more like a <blank> token. 

Finally, remember that we need to sum all the probability of different paths but with respect to the same prefix together. 
From line 73 to 75, if it turns out that `l_plus` is not in `A_prev`, it indicates that we may have discarded this prefix at the last time step. 
To recover them, we need simply do as shown in line 74 and 75. On the other hand, if `l_plus` is still in `A_prev`, 
it means we still focus on it and all the paths regarding it have been taken into account. 
You may have some trouble with line 75 as it only contains `Pnb[t-1][l_plus]` but without `Pb[t-1][l_plus]`. 
However, it is obvious that if `l_plus` ends with <blank> token and after adding a new non-blank token, this will finally change `l_plus` into a new prefix completely. 

At last, from line 79 to 81, we firstly extracted all the prefixes we are focusing on and sort them according to the `sorter`, 
where we consider both the CTC probability and the length of the word sequences with a hyperparameter `beta`. 
Thus, now we know that `beta`, the so-called compensation, is the hyperparameter to control the output length. 
A larger `beta` would encourage the model to output longer sequences. 
Finally, the output would be `A_prev[0].strip('>')`. 
