---
layout: post
title: "Docker for ML Model Deployment: From Dev Chaos to Reproducible Builds"
comments: true
description: "How Docker solved dependency hell for building TTS and OCR models with Android NDK and TensorFlow at Indus OS"
keywords: "docker, ml deployment, android ndk, tensorflow, reproducible builds, ci cd, indus os"
---

At Indus OS, our ML pipeline touched TensorFlow (Python 2.7, specific commit hash), Android NDK r12b, Bazel 0.3.1, and custom C libraries for the TTS vocoder. If any version drifted, the build broke silently and produced models that crashed on device. Three engineers on the team, three different Ubuntu versions, three different sets of installed libraries. Docker fixed this.

## The Dependency Problem

Building our TensorFlow Android library required exact versions of everything. NDK r12b worked. NDK r13 introduced a libc++ change that broke our NEON-optimized matrix multiply. Bazel 0.3.2 changed a flag that affected selective op registration. We lost two days to each of these issues before we started using Docker.

```dockerfile
FROM ubuntu:14.04

# Exact versions that produce working builds
ENV NDK_VERSION=r12b
ENV BAZEL_VERSION=0.3.1
ENV TF_COMMIT=a23f5d7

RUN apt-get update && apt-get install -y \
    build-essential git python2.7 python-pip wget unzip openjdk-8-jdk

# Android NDK
RUN wget -q https://dl.google.com/android/repository/android-ndk-${NDK_VERSION}-linux-x86_64.zip \
    && unzip -q android-ndk-${NDK_VERSION}-linux-x86_64.zip -d /opt \
    && rm android-ndk-${NDK_VERSION}-linux-x86_64.zip
ENV ANDROID_NDK_HOME=/opt/android-ndk-${NDK_VERSION}

# Bazel
RUN wget -q https://github.com/bazelbuild/bazel/releases/download/${BAZEL_VERSION}/bazel-${BAZEL_VERSION}-installer-linux-x86_64.sh \
    && chmod +x bazel-${BAZEL_VERSION}-installer-linux-x86_64.sh \
    && ./bazel-${BAZEL_VERSION}-installer-linux-x86_64.sh

# TensorFlow at exact commit
RUN git clone https://github.com/tensorflow/tensorflow.git /tf \
    && cd /tf && git checkout ${TF_COMMIT}

WORKDIR /tf
```

## Images, Containers, and Volumes for Model Files

The key concepts that mattered for us:

An image is a frozen snapshot of the build environment. We built it once and pushed it to our private registry. Every engineer and the CI server pulled the same image. No more "works on my machine."

A container is a running instance of that image. We ran builds inside containers and extracted the output artifacts.

Volumes let us mount model weight files and training data without baking them into the image. Models changed daily. The build environment changed maybe once a month.

```bash
# Build the TF Android library inside the container
docker run --rm \
  -v $(pwd)/models:/models \
  -v $(pwd)/output:/output \
  indus-ml-build:latest \
  bash -c "cd /tf && bazel build -c opt \
    --copt='-DSELECTIVE_REGISTRATION' \
    //tensorflow/contrib/android:libtensorflow_inference.so \
    && cp bazel-bin/tensorflow/contrib/android/*.so /output/"
```

## CI/CD for the ML Pipeline

Before Docker, our "CI" was someone running the build on their laptop and copying the .so file to a shared drive. With Docker, we set up a proper pipeline:

1. Train model in Python container, export frozen graph
2. Quantize and optimize in TF tools container
3. Build Android .so with NDK container
4. Run on-device tests via ADB from the same container

```bash
#!/bin/bash
# build_pipeline.sh

# Step 1: Freeze and quantize model
docker run --rm -v $(pwd):/work indus-ml-train:latest \
  python /work/scripts/freeze_and_quantize.py \
    --input_model /work/checkpoints/latest \
    --output /work/artifacts/quantized_model.pb

# Step 2: Build Android library with model baked in
docker run --rm -v $(pwd):/work indus-ndk-build:latest \
  ndk-build -C /work/android_app APP_ABI=armeabi-v7a

echo "Build artifacts in ./artifacts/"
```

Each step used a different image with only the tools it needed. The train image had Python and TF. The NDK image had the Android toolchain. No cross-contamination.

## What Worked and What Didn't

Docker eliminated environment drift completely. Build times went from "2 hours plus debugging" to "45 minutes, deterministic." New engineers could build on day one instead of spending a week setting up their environment.

What didn't work: Docker on macOS was painfully slow for our builds because of the filesystem virtualization layer. We moved CI to a Linux server and kept macOS only for development. Also, the Docker images were large (2.5GB for the full TF build environment). We eventually used multi-stage builds to keep the final images smaller, but the build stage image stayed big. Storage was cheap. Engineer time was not.
