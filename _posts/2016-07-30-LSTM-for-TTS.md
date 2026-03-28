---
layout: post
title: "LSTM-Based Text to Speech for 6 Indian Languages on Android NDK"
comments: true
description: "Replacing HMM-based TTS with LSTM on Android NDK in C for Hindi, Tamil, Telugu, Malayalam, Bengali, and Marathi"
keywords: "tts, lstm, android ndk, indic languages, hindi, tamil, telugu, speech synthesis, hmm"
---

Indus OS needed TTS for 6 Indian languages: Hindi, Tamil, Telugu, Malayalam, Bengali, and Marathi. The existing HMM-based system sounded robotic and handled prosody poorly. We replaced it with an LSTM-based pipeline running entirely on-device in C through Android NDK. The constraint: real-time synthesis on a single CPU core with under 100MB memory.

## The Two-Model Architecture

Our pipeline had two models. A duration model predicted how long each phoneme should last. An acoustic model generated spectral features (mel-cepstral coefficients + F0) frame by frame. Both were 2-layer LSTMs with 256 hidden units.

```
Text -> Phoneme Conversion -> [Duration Model] -> [Acoustic Model] -> Vocoder -> Audio
```

The duration model took phoneme-level linguistic features as input: phoneme identity, position in syllable, syllable stress, word position in phrase, and phrase-level features. For Hindi alone this was a 342-dimensional binary feature vector plus 25 numerical features, following the Zen/Senior/Schuster architecture.

```c
// Phoneme feature extraction (simplified)
typedef struct {
    int phoneme_id;        // one-hot encoded, 48 classes for Hindi
    int pos_in_syllable;   // forward/backward position
    int syllable_stress;   // 0 or 1
    int num_syllables_in_word;
    int word_pos_in_phrase;
    float duration_sec;    // output from duration model
} PhonemeFeature;
```

The acoustic model took the duration-expanded features and predicted 60-dimensional output per frame: 25 mel-cepstral coefficients, 25 delta coefficients, log F0, delta log F0, and voiced/unvoiced flag. We used the MLSA (Mel Log Spectrum Approximation) vocoder to convert these features to audio.

## Running LSTM Inference in C

We wrote the LSTM forward pass from scratch in C. No TensorFlow, no framework overhead. Just matrix multiplications and sigmoid/tanh activations. The weight matrices were exported from our Python training code as flat binary files.

```c
// LSTM cell forward pass
void lstm_forward(LSTMCell *cell, float *input, float *h_prev,
                  float *c_prev, float *h_out, float *c_out) {
    int n = cell->hidden_size;
    float gates[4 * n];

    // input, forget, cell, output gates
    mat_mul(cell->W_ih, input, gates, 4*n, cell->input_size);
    mat_mul_add(cell->W_hh, h_prev, gates, 4*n, n);
    vec_add(gates, cell->bias, gates, 4*n);

    for (int i = 0; i < n; i++) {
        float ig = sigmoid(gates[i]);
        float fg = sigmoid(gates[n + i]);
        float cg = tanhf(gates[2*n + i]);
        float og = sigmoid(gates[3*n + i]);
        c_out[i] = fg * c_prev[i] + ig * cg;
        h_out[i] = og * tanhf(c_out[i]);
    }
}
```

Each language had its own weight files but shared the same inference code. Total model size per language was about 3.2MB (quantized to 16-bit floats). All 6 languages together: 19.2MB on disk.

## Latency and Memory on Real Devices

Our target was synthesis faster than 2x real-time. That means 1 second of audio should take less than 2 seconds to generate. On a Mediatek MT6582 at 1.3GHz (single core), we measured:

| Metric | Value |
|--------|-------|
| Duration model inference | 12ms per utterance |
| Acoustic model inference | 1.4ms per frame |
| Vocoder (MLSA) | 0.8ms per frame |
| Total for 3-second utterance | ~1.8 seconds |
| Peak memory usage | 42MB |

The bottleneck was the acoustic model. At 200 frames/second (5ms frame shift), a 3-second utterance needed 600 forward passes. We optimized the matrix multiply with NEON intrinsics which gave us a 2.3x speedup on ARM.

## Language-Specific Pitfalls

Each language had its own phoneme set and grapheme-to-phoneme rules. Tamil has no aspirated consonants. Malayalam has a complex sandhi system where phonemes change at word boundaries. Bengali vowel nasalization required special handling in the feature extraction.

We maintained separate G2P (grapheme-to-phoneme) rule files per language, written as finite state transducers in C. The Hindi G2P alone had 180 rules. Getting these right was more work than the neural network itself.

## What Worked and What Didn't

Writing inference in pure C eliminated framework overhead entirely. On a 512MB device, 42MB peak memory was acceptable. The LSTM output sounded noticeably more natural than HMM, especially for Hindi and Tamil where we had the most training data (8 hours each).

What didn't work: we tried sharing a single multilingual model across all 6 languages. Quality dropped significantly for Malayalam and Bengali which have very different prosody patterns. Separate models per language was the only way to maintain quality. We also tried GRU cells to save compute but the quality difference was audible, so we stuck with LSTM.
