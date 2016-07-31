---
layout: post
title: "(Incomplete) Exploring LSTM's for Text to speech (TTS) applications"
comments: true
description: "Use of deep learning techniques to replace the traditional HMM based TTS systems"
keywords: "deep learning, tts, LSTM, HMM, tensorflow, android"
---

###Think : Serving tensorflow based LSTM model on android for TTS application


###Learn: Basics of LSTM 

**Architecture**

####Duration model

####Input Features

Phoneme-level linguistic features?  

```
**STATISTICAL PARAMETRIC SPEECH SYNTHESIS USING DEEP NEURAL NETWORKS** 
by _Heiga Zen, Andrew Senior, Mike Schuster_

The input features for the DNN-based systems included 342 bi- nary features for categorical linguistic contexts (e.g. phonemes iden- tities, stress marks) and 25 numerical features for numerical linguis- tic contexts (e.g. the number of syllables in a word, position of the current syllable in a phrase). In addition to the linguistic contexts- related input features, 3 numerical features for coarse-coded posi- tion of the current frame in the current phoneme and 1 numerical feature for duration of the current segment were used.

We also tried to encode numerical features to binary ones by applying questions such as “is-the-number-of-words-in-a-phrase-less-than-5”. A pre- liminary experiment showed that using numerical features directly worked better and more efficiently than encoding them to binary ones.
````

```
**AN HMM-BASED SPEECH SYNTHESIS SYSTEM APPLIED TO ENGLISH**
by _Keiichi Tokuda, Heiga Zen, Alan W. Black_

phoneme:
- preceding, current, succeeding phoneme
- position of current phoneme in current syllable

syllable:
- number of phonemes at preceding, current, succeeding
- accent of preceding, current, succeeding syllable
- stress of preceding, current, succeeding syllable
- position of current syllable in current word syllable
- number of preceding, succeeding stressed syllables in current phrase
- number of preceding, succeeding accented syllables in current phrase stressed syllable accented syllable
- number of syllables
- number of syllables
- vowel within current syllable

word:
- guess at part of speech of preceding, current, succeeding word
- number of syllables in preceding, current, succeeding word
- position of current word in current phrase
- number of preceding, succeeding content words in current phrase
- number of words from previous, to next content word

phrase:
- number of syllables in preceding, current, succeeding phrase
- position in major phrase
- ToBI endtone of current phrase utterance:
- number of syllables in current utterance

utterance:
- number of syllables in current utterance
```

####Output Features

```
**STATISTICAL PARAMETRIC SPEECH SYNTHESIS USING DEEP NEURAL NETWORKS** 
by _Heiga Zen, Andrew Senior, Mike Schuster_

The out- put features were basically the same as those used in the HMM- based systems. To model logF0 sequences by a DNN, the con- tinuous F0 with explicit voicing modeling approach [37] was used; voiced/unvoiced binary value was added to the output features and log F0 values in unvoiced frames were interpolated. To reduce the computational cost, 80% of silence frames were removed from the training data.
```

How to compute the phoneme level duratons?


####Acoustic model
####Input Features

####Output Features

###Code: 
**Tensorflow graphs for modeling LSTM**

**Serve models to consumers**

**Limitations**

**Refernces**