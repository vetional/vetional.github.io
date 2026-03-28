---
layout: post
title: "Building OCR for Indic Languages on Budget Android Phones"
comments: true
description: "CNN-based character recognition for Hindi and Devanagari on devices with no GPU and under 512MB RAM"
keywords: "ocr, indic languages, hindi, devanagari, cnn, android ndk, mobile ml, tensorflow"
---

Indus OS needed on-device OCR for Hindi and other Indic scripts. The phones had no GPU, 512MB RAM, and Mediatek SoCs running at 1.3GHz. Cloud OCR was not an option because most users were on 2G connections with 2-5 second round trips.

## The Indic Script Problem

English has 26 uppercase and 26 lowercase characters. Hindi (Devanagari) has 47 base characters, but conjunct consonants push the effective class count past 400. For example, क + ष = क्ष is a single visual glyph. Tamil and Telugu have similar combinatorial explosions.

We handled this by decomposing recognition into two stages. First, detect individual character components (base consonant, vowel mark, conjunct). Second, combine them using script-specific rules. This brought our output classes down to ~120 per language instead of 400+.

## A Small CNN That Fits

We needed the model under 2MB after quantization. Three convolutional layers with small filters got us there.

```python
import tensorflow as tf

def build_ocr_model(num_classes=120):
    inp = tf.placeholder(tf.float32, [None, 32, 32, 1], name='input')

    # 3 conv layers, small filters, no pooling after last
    x = tf.layers.conv2d(inp, 32, 3, activation=tf.nn.relu, padding='same')
    x = tf.layers.max_pooling2d(x, 2, 2)
    x = tf.layers.conv2d(x, 64, 3, activation=tf.nn.relu, padding='same')
    x = tf.layers.max_pooling2d(x, 2, 2)
    x = tf.layers.conv2d(x, 64, 3, activation=tf.nn.relu, padding='same')

    x = tf.layers.flatten(x)
    x = tf.layers.dense(x, 128, activation=tf.nn.relu)
    out = tf.layers.dense(x, num_classes, name='output')
    return inp, out
```

After quantization to 8-bit, the model was 1.8MB. Inference on a Mediatek MT6582 (single core, no NEON optimizations) took 85ms per character crop. With NEON enabled via NDK compiler flags, it dropped to 62ms.

```bash
# NDK build with NEON
ndk-build APP_ABI=armeabi-v7a \
  LOCAL_CFLAGS="-mfpu=neon -mfloat-abi=softfp"
```

## Training Data for Low-Resource Languages

This was the hardest part. We had no large labeled dataset for printed Devanagari. We generated synthetic training data by rendering text in 15 common Hindi fonts at various sizes, rotations, and noise levels. This gave us ~500K training samples per language.

For Tamil and Telugu we had even fewer fonts available. We augmented aggressively: elastic distortions, Gaussian blur, salt-and-pepper noise. The augmentation pipeline ran on a single GPU machine overnight and produced 200K samples per language.

```python
import cv2
import numpy as np

def augment_char(img):
    # Random rotation within 5 degrees
    angle = np.random.uniform(-5, 5)
    M = cv2.getRotationMatrix2D((16, 16), angle, 1.0)
    img = cv2.warpAffine(img, M, (32, 32))
    # Gaussian noise
    noise = np.random.normal(0, 10, img.shape).astype(np.uint8)
    return cv2.add(img, noise)
```

## Constraints on Low-Memory Devices

We processed one character at a time, never batching. Batching would spike memory and trigger the Android low-memory killer. On a 512MB device, if your app crosses ~80MB, the OS will kill background processes. Cross ~120MB and your own app gets killed.

We also avoided loading the model at app startup. Instead we loaded it lazily on first OCR request and kept a weak reference. If the system reclaimed the memory, we reloaded on next use. This added 200ms latency on a cold start but kept the app alive.

## What Worked and What Didn't

Synthetic data generation worked surprisingly well for printed text. We hit 94% character accuracy on Hindi, 91% on Tamil. Handwritten recognition was a different story. We tried it and accuracy dropped to 68%, so we scoped it out.

The two-stage decomposition approach was essential. A single classifier over 400+ classes never converged well with our limited training data. Breaking it into components made the problem tractable.
