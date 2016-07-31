---
layout: post
title: "(Incomplete) Tensorflow basics"
comments: true
description: "Graphs, distributed computations, protobuf and tensorflow serving"
keywords: "basics of tensorflow, serving, distributed"
---

**Disclaimer: Most of the writeup may match verbatim to the tensorflow docs, infact the idea of this article is to takeout the most relevant parts of the tensorflow doc and present them in a simple manner.**


#### **Think: Serving tensorflow on android apps a broad overview**

A unified environment to model and distribute your learned models on Andorid.

#### **Learn: Basics of tensorflow** 

Setting up tensorflow from [source](https://gist.github.com/vetional/3f75fa1a0a3923912d7b58819abef29f)

**What are computation Graphs**

TensorFlow programs are usually structured into a construction phase, that assembles a graph, and an execution phase that uses a session to execute ops in the graph.

Tensorflow represents every computation as a graph. (why graphs?)

The foundation of computation in TensorFlow is the Graph object. This holds a network of nodes, each representing one operation, connected to each other as inputs and outputs.

The protobuf tools parse this text file, and generate the code to load, store, and manipulate graph definitions. If you see a standalone TensorFlow file representing a model, it's likely to contain a serialized version of one of these GraphDef objects saved out by the protobuf code. There are actually two different formats that a ProtoBuf can be saved in text and binary. The API itself can be a bit confusing - the binary call is actually ParseFromString(), whereas you use a utility function from the text_format module to load textual files.

Once you've loaded a file into the graph_def variable, you can now access the data inside it. For most practical purposes, the important section is the list of nodes stored in the node member. Each node is a NodeDef object, also defined in graph.proto. These are the fundamental building blocks of TensorFlow graphs, with each one defining a single operation along with its input connections. Here are the members of a NodeDef, and what they mean.

**Freezing Graphs**

One confusing part about this is that the weights usually aren't stored inside the file format during training. Instead, they're held in separate checkpoint files, and there are Variable ops in the graph that load the latest values when they're initialized. It's often not very convenient to have separate files when you're deploying to production, so there's the freeze_graph.py script that takes a graph definition and a set of checkpoints and freezes them together into a single file.

What this does is load the GraphDef, pull in the values for all the variables from the latest checkpoint file, and then replace each Variable op with a Const that has the numerical data for the weights stored in its attributes It then strips away all the extraneous nodes that aren't used for forward inference, and saves out the resulting GraphDef into an output file.

**NOTE**
In TensorFlow, the filter weights for the Conv2D operation are stored on the second input, and are expected to be in the order [filter_height, filter_width, input_depth, output_depth], where filter_count increasing by one means moving to an adjacent value in memory.

**How to compute with graphs**

As I said, TensorFlow programs are usually structured into a construction phase, that assembles a graph, and an execution phase that uses a session to execute ops in the graph.

So instead of showing you the code for it let's first see a visualization for linear regression:


**Serve models to consumers**

**Limitations**

#### Code