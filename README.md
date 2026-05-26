# Continuous Keyframe-Free Holographic Compression for Real-Time Streaming

An innovative C++ paradigm for real-time audio-video streaming designed to deliver stable $320\times240$ resolution at 15 fps over genuine, ultra-low bandwidth 2G GSM networks with high latency and heavy packet loss.
[Download Full Technical Paper (PDF)](STREAMING2g.pdf)

## 📋 System Prerequisites

Before running the binaries, ensure your environment meets the following requirements:

* **Operating System:** Windows 11 (64-bit) or Windows 10 (64-bit).
* **FFmpeg Dependency:** FFmpeg must be installed on your system and added to your system's **PATH** environment variable. The framework relies on FFmpeg for underlying media handling and streaming pipelines.
  * *Verification:* You can verify your installation by opening a command prompt and typing `ffmpeg -version`.

---

## 📺 Live Video Demonstration

[![Watch the 2G Holographic Streaming Demo](https://img.youtube.com/vi/2jCk43ZhJ_0/maxresdefault.jpg)](https://www.youtube.com/watch?v=2jCk43ZhJ_0)

---

## 📝 Abstract

This paper presents a real-time audio-video streaming system capable of delivering stable $640\times480$ resolution at 25 fps with synchronized audio over genuine 2G GSM networks (9-20 kbps, high packet loss, high latency). 

The proposed method departs from conventional predictive codecs by eliminating keyframes entirely and introducing a continuous holographic compression model based on high-dimensional geometric embedding, adaptive multi-view projection, and sparse reconstruction. 

The system exhibits intrinsic resilience to packet loss and bandwidth degradation, achieving graceful quality scaling instead of freezing or collapse. Experimental observations further indicate that Forward Error Correction (FEC) is unnecessary and may reduce effective quality in this regime.

---

## 🧠 Core Innovation

Unlike conventional predictive codecs (H.264/H.265) that suffer from extreme fragility due to keyframes and inter-frame dependencies, this framework introduces a **Continuous Holographic Compression Model**. By mapping patches into a high-dimensional latent space via geometric embedding and leveraging sparse reconstruction (OMP over DCT dictionary), the stream exhibits intrinsic resilience to data loss. 

Instead of freezing or collapsing abruptly, the signal achieves **graceful, continuous quality scaling**.

### Key System Properties:
* **Keyframe-Free Architecture:** Eliminates propagation errors and structural stream collapse.
* **Intrinsic Robustness:** Information is non-localized and redundantly distributed across projection operators; missing packets result only in minor, localized degradation.
* **FEC-Free Optimization:** Demonstrates that Forward Error Correction reduces effective signal quality in extreme bandwidth regimes, proving that raw, sparse geometric data maximizes throughput.

## 🛠️ System Pipeline & Architecture

1. **Acquisition:** Real-time frame capture via OpenCV / FFmpeg.
2. **Mathematical Processing:** High-dimensional embedding $\phi:\mathbb{R}^{n}\rightarrow\mathbb{R}^{D}$ and adaptive multi-view projection.
3. **Sparse Coding:** Orthogonal Matching Pursuit (OMP) optimized over a discrete cosine transform (DCT) dictionary.
4. **Transport:** Zstandard entropy compression, UDP packetization, and RTSP/WebRTC relaying via MediaMTX.

---

## 💻 Quick Start & Usage Examples

The framework uses a unified CLI binary `mio_codec_cli_opencv_full.exe` for both sending and receiving video and audio streams via UDP.

### 1. Video Receiver Setup
To start listening for an incoming video stream on port 5000 and launch the local web server component:

mio_codec_cli_opencv_full.exe stream-recv-webcam-full udp://0.0.0.0:5000 out_webdir=web "opts=http_port=8080,mtu=1200,profile=medium,xdel=20,ydel=3,fec=off"

### 2. Video Sender Setup
To capture from your webcam and stream to the receiver's IP address (replace 1.2.3.4 with your target IP):

mio_codec_cli_opencv_full.exe stream-send-webcam-full udp://1.2.3.4:5000 "opts=w=320,h=240,fps=20,mtu=1200,profile=medium,dur=0,xdel=20,ydel=3,fec=off"

### 3. Audio Receiver Setup
To listen for incoming synchronized audio streams on port 5001:

mio_codec_cli_opencv_full.exe stream-recv-audio udp://0.0.0.0:5001

### 4. Audio Sender Setup
To capture audio from your local hardware microphone and stream it to the receiver:

mio_codec_cli_opencv_full.exe stream-send-audio udp://1.2.3.4:5001 "Gruppo microfoni (Realtek(R) Audio)"

*Note: For single-machine local testing, you can loop back the traffic or use an external UDP port-forwarding/remap utility such as portmap.io.*

---

## ⚙️ Advanced Parameter Tuning (opts)

You can fine-tune the holographic compression pipeline using the comma-separated options string:

| Parameter | Allowed Values | Description |
| :--- | :--- | :--- |
| profile | extra, low, medium | Controls processing depth and resource allocation profiles. |
| xdel | Integer (e.g., 20) | Frame-change percentage threshold to trigger a complete geometric patch refresh. Lowering this value improves visual accuracy but increases bandwidth requirements. |
| ydel | Integer (e.g., 3) | Threshold percentage used to flag a frame area as "unchanged". Lowering this parameter improves fluid motion rendering at the cost of higher bandwidth. |
| fec | on \| off | Toggles Forward Error Correction for damaged packet recovery. Recommended: off. As demonstrated in the paper, disabling FEC maximizes effective network throughput and signal consistency under extreme bandwidth constraints. |
| mtu | Integer (e.g., 1200) | Maximum Transmission Unit size for UDP packetization to prevent fragmentation over legacy links. |

---

## 🤝 Commercial Inquiries & Collaborations
This work is part of a potential patent disclosure. For inquiries regarding commercial licensing, production-grade implementations, or research collaborations, please contact the author at:
xdaniele.rufox@gmail.com

---
*© 2026 - Continuous Keyframe-Free Holographic Compression 
---

---

### Support my Research 🚀
If you find this project useful for your benchmarks or academic evaluation, consider supporting my independent research:

[![Donate with PayPal](https://img.shields.io/badge/Donate-PayPal-blue.svg)](https://paypal.me/xdanielex272)
[![Donate with BTC](https://img.shields.io/badge/Donate-Bitcoin-orange.svg)](#)
[![Donate with USDT](https://img.shields.io/badge/Donate-Tether-green.svg)](#)

* **Bitcoin (BTC):** `bc1q4l9v8welwr6mp4g6uc2t7ex0n274malynq6yqj`
* **Tether (USDT - TRC20):** `TA3m7pqk1mTgZtFQHf7KufAqnaqsN95kPh`

---
