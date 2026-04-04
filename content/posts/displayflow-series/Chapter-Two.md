---
title: "Chapter Two: The Pipe Bridge"
date: 2026-03-04
tags: ["IPC", "Rust", "Architecture"]
description: "Decoupling Engine from the frontend using Windows Named Pipes."
---

## Decoupling the Engine

To keep the Engine lightweight and headless, I’ve implemented a **Named Pipe** architecture. This allows an external binary frontend to communicate with the core Rust service without the overhead of a full networking stack or the complexity of shared memory.

### Why Named Pipes?

In the context of a system tool like **displayflow**, pipes offer several strategic advantages:

* **Low Latency**: Faster than local HTTP or RPC-over-network for simple signaling.
* **Security**: Leveraging native Windows Access Control Lists (ACLs) to ensure only authorized processes can trigger display changes.
* **Zero-Footprint**: No need to open firewall ports or manage local IP bindings.

### The Communication Flow

The engine acts as the pipe server, listening for structured commands from the frontend. This decoupling is essential for the upcoming **CCD API** (Connecting and Configuring Displays) implementation, as it allows the hardware logic to remain isolated from the UI state.

### Architecture Preview

As seen in the project's GitHub repository, I have already prepared the "pipe" to handle incoming display tasks. This bridge ensures that even if the frontend crashes, the `DFEngine` remains stable, holding the last known good configuration in the GDI staging area.

In the next post, I will break down the asynchronous message loop that processes these external triggers and converts them into the `stage_config` calls we discussed in Chapter One.