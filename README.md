# Continuous Keyframe-Free Compression for Real-Time Streaming

A C++ codec for real-time audio-video streaming engineered to deliver a **stable, smooth feed over genuine ultra-low-bandwidth 2G / EDGE networks** and high-cost satellite uplinks — links with high latency and heavy packet loss where conventional predictive codecs fail.

[Download the full tech paper (PDF)](STREAMING2g.pdf)

---

## ✨ What's new in this version

* **End-to-end encryption, video *and* audio.** Both channels can now be protected with **ChaCha20-Poly1305** authenticated encryption (RFC 8439), implemented from scratch with no external crypto dependency — validated against the official RFC 8439 / FIPS 180-4 test vectors. Enable it with a single shared passphrase (`key=...`) on each channel you use; without it, streams are sent exactly as before (fully backward compatible). A frame that fails authentication (wrong key, or tampering) is dropped rather than decoded — no partial/corrupted leakage.
* **MediaMTX dependency removed.** The receiver used to auto-launch MediaMTX and re-encode every decoded frame to H.264/RTSP for a browser preview — on modest hardware this extra real-time encode was, in testing, the actual CPU bottleneck once encryption and audio were both active, not the encryption itself. It has been replaced with a **native local preview window** (OpenCV `imshow`), opened automatically by the receiver: no external process, no re-encoding, effectively free on the CPU. On a machine with no display attached (e.g. a headless Linux box) it degrades gracefully — the window is simply skipped, everything else keeps working.
* **Modular source layout.** The implementation is now split across focused files (`common.hpp`, `transport.hpp`, `audio.hpp`, `codec_core.hpp`, `crypto.hpp`/`crypto.cpp`, `bandwidth_monitor.hpp`, `shell_desktop.cpp`) instead of one monolithic translation unit. This was a mechanical, behavior-preserving split — verified line-for-line against the previous single-file version before any new feature (encryption) was added on top — meant to make the codebase auditable and easier to extend safely.
* **Genuine colour** (carried over from the previous release). Chrominance is a low-resolution plane sent only when it actually changes, so a static background keeps its colour at almost no cost. On the **medium** profile it delivers true-colour audio-video that stays fluid on a real 2G/EDGE link.

---

## 📋 System Prerequisites

* **Operating System:** Windows 10/11 (64-bit) **or Linux** (64-bit) — both are now first-class, actively verified targets; see [Build](#-build) for the exact toolchain and dependencies for each.
* **[OpenCV](https://opencv.org/)** — C++ headers and a prebuilt library (`opencv_world4xx.lib` on Windows, `libopencv-dev` on Linux).
* **[zstd](https://facebook.github.io/zstd/)** — the entropy-compression library.
* **FFmpeg / ffplay / ffprobe** — used for capture (DirectShow on Windows, V4L2 on Linux), audio (Opus/OGG), decoding, and playback. Must be on your **PATH**.
* ~~MediaMTX~~ — **no longer used**, see *What's new* above.

---

## 📝 Abstract

A real-time audio-video streaming system for genuine low-bandwidth degraded links — 2G/EDGE cellular and high-cost satellite uplinks — where conventional predictive codecs fail. The design departs from the classic keyframe + inter-frame predictive model: frames are encoded as a sequence of **patch-level modes** that are autonomously decodable, so no picture depends on a long chain of earlier pictures. The result is a stream that **degrades gracefully under packet loss instead of freezing** — when fragments are dropped the effective frame rate dips and then recovers, but the transmission never stalls.

The representation follows the *holographic* principle that information should be distributed rather than localized, so that partial or imperfect delivery still yields a usable picture. In practice the codec realizes this as a sparse, motion-adaptive, keyframe-free pipeline built on per-patch temporal differencing, adaptive quantization, run-length coding and Zstandard entropy compression, with an efficient colour extension. It targets the operational regimes that matter for **surveillance, unmanned aerial platforms and telemedicine** — where continuous, reliable awareness matters more than resolution.

---

## 🧠 Core Innovation

Unlike conventional predictive codecs (H.264/H.265), which are fragile because a lost inter-frame corrupts everything until the next keyframe, this framework uses a **continuous, motion-adaptive, keyframe-free patch coder**. Each picture is reduced to a working patch; for every patch the encoder takes a three-way decision:

* **skip** — unchanged within a threshold: nothing is transmitted, so losing its packet cannot corrupt the picture,
* **delta** — the residual is quantized with an adaptive scale and transmitted (the steady-state low-cost path),
* **intra** — a full patch is sent, refreshing the reference and bounding any accumulated error.

This yields the properties that matter on a lossy, high-latency channel:

* **Keyframe-free architecture.** No mandatory periodic reference frame; the frame rate adapts to the link.
* **Graceful, continuous degradation.** Quality scales instead of collapsing.
* **Immediate recovery.** A lost frame is repaired by the next complete frame.
* **Intrinsic robustness.** Skipped patches transmit no information, so loss cannot produce a stream stall.
* **FEC-free.** Forward Error Correction spends scarce bits that reduce effective quality; in this regime it is better left off.

### System pipeline

1. **Acquisition** — real-time frame capture via OpenCV / FFmpeg.
2. **Patch processing** — reduce to a working patch and take the per-patch mode decision (skip / delta / intra) against a local reference.
3. **Adaptive quantization** — per-frame scale from a robust (MAD) estimate of the residual.
4. **Entropy coding** — run-length encoding + Zstandard compression.
5. **Colour** — a low-resolution chroma plane, sent only when it changes, upscaled and saturation-boosted.
6. **Encryption** *(optional, opt-in)* — ChaCha20-Poly1305 authenticated encryption of the whole encoded blob, keyed from a shared passphrase.
7. **Transport** — UDP packetization with fragmentation and a checksum, reassembled on the receiver.
8. **Delivery** — a native local preview window on the receiving machine (no external relay process required).

---

## 🌐 Network model: how two clients actually connect (read this before deploying)

This version uses **direct UDP** between the two endpoints — there is **no NAT-traversal or rendezvous server yet**. Concretely, that means:

* The **receiver** binds a UDP port on `0.0.0.0` (listens on all interfaces) and must be **reachable** at the address the sender is told to use.
* The **sender** sends directly to that address:port — no discovery, no signalling, nothing in between.

**What "reachable" requires in practice:**

* If both machines are on the **same local network** (same Wi‑Fi/LAN), use the receiver's local IP (e.g. `192.168.1.20`) — this always works, no further setup needed.
* If the two machines are on **different networks** (the realistic case for two remote operators, e.g. over cellular data), the receiver needs to be reachable from the public internet. In this version that means one of:
  * the receiver machine has a **public IP address** directly on its network interface, or
  * the receiver is behind a router where you can configure **port forwarding** for the UDP port in use, or
  * you use a third-party UDP port-mapping/tunnelling service (e.g. [portmap.io](https://portmap.io)) to expose the receiver's port publicly for testing.
* Most residential and virtually all mobile/cellular connections sit behind **carrier-grade NAT**, which none of the above can bypass on the sender's side by itself — it is the **receiver** side that needs the public reachability; the sender can be behind any ordinary NAT.

A dedicated rendezvous/relay server (so that **neither** side needs a public IP — both simply register with a shared call code and the relay mirrors traffic between them) is architecturally straightforward to add on top of this transport and is the natural next step for two mobile clients calling each other with no manual network configuration; it is **not included in this version**.

---

## 💻 Quick Start & Usage

The framework uses a unified CLI binary for sending and receiving video and audio streams over UDP.

### 1. Video Receiver

```bat
mio_codec_cli_opencv_full_color.exe stream-recv-webcam-full udp://0.0.0.0:5000 out_webdir=web "opts=http_port=8080,mtu=1200,profile=medium,xdel=20,ydel=3,fec=off,key=my-shared-passphrase"
```

### 2. Video Sender

```bat
mio_codec_cli_opencv_full_color.exe stream-send-webcam-full udp://1.2.3.4:5000 "opts=w=320,h=240,fps=20,mtu=1200,profile=medium,dur=0,xdel=20,ydel=3,fec=off,key=my-shared-passphrase"
```

> `key=<passphrase>` enables ChaCha20-Poly1305 encryption of the video channel. Omit it (on both sides) for an unencrypted stream, exactly as before. The two sides must use the **identical** passphrase — a mismatch (or a missing key on one side) causes that side to drop every frame rather than decode garbage.

### 3. Standalone Audio Receiver (RTP)

```bat
mio_codec_cli_opencv_full_color.exe stream-recv-audio udp://0.0.0.0:5001 key=my-shared-passphrase
```

### 4. Standalone Audio Sender (RTP)

```bat
mio_codec_cli_opencv_full_color.exe stream-send-audio udp://1.2.3.4:5001 "Gruppo microfoni (Realtek(R) Audio)" key=my-shared-passphrase
```

> Audio travels on its own independent UDP channel/port, deliberately decoupled from the video channel — this keeps a hiccup on one channel from affecting the other. `key=` is **optional and independent** of the video channel's key: there is no shared state between the two processes (they are separate programs), so encrypting audio simply means passing the same `key=` value to both audio commands, exactly as you already do for the two video commands. Omit `key=` entirely to keep the original, unencrypted native-RTP audio path unchanged.

*Note: for single-machine local testing you can loop the traffic back (`udp://127.0.0.1:PORT`), or use an external UDP port-forwarding/remap utility such as portmap.io — see [Network model](#-network-model-how-two-clients-actually-connect-read-this-before-deploying) above.*

---

## 🔧 Build

### Windows (MSVC)

Open an **x64 Native Tools Command Prompt for VS** and run (adjust the OpenCV/zstd paths if yours differ):

```bat
cl /EHsc /std:c++17 shell_desktop.cpp crypto.cpp ^
   /I "C:\opencv\build\include" /I "C:\path\to\zstd" ^
   /link /LIBPATH:"C:\opencv\build\x64\vc15\lib" /LIBPATH:"C:\path\to\zstd" ^
   opencv_world455.lib zstd.lib Ws2_32.lib Advapi32.lib ^
   /Fe:mio_codec_cli_opencv_full_color.exe
```

`crypto.cpp` is a new required source file (the ChaCha20-Poly1305/SHA-256 implementation); `Advapi32.lib` is a new required system library (used for `CryptGenRandom`, the secure random source for session keys).

### Linux (g++)

```bash
sudo apt-get install -y build-essential libopencv-dev libzstd-dev pkg-config
g++ -std=c++17 -O2 shell_desktop.cpp crypto.cpp -o mio_codec_cli_opencv_full_color \
    $(pkg-config --cflags --libs opencv4) -lzstd -lpthread
```

> **Runtime dependencies (Linux):** the binary links dynamically against OpenCV (which on most distributions pulls in Qt and GStreamer as well) and `libzstd`; `ffmpeg`/`ffplay`/`ffprobe` are still needed in `PATH`. Installing `libopencv-dev` on the machine that *runs* the binary (not only the one that compiled it) is the simplest way to satisfy all of it. A display (X11/Wayland) is only needed for the local preview window — without one, the receiver logs a notice and continues without it.

> **Important:** sender and receiver must both run the same binary / format. This build's block header differs from older single-file releases, so a stream encoded with this build must be decoded by this build.

---

## ⚙️ Advanced Parameter Tuning (`opts`)

| Parameter | Allowed values | Description |
| :--- | :--- | :--- |
| `w`, `h` | int (e.g. `320`, `240`) | Capture resolution. |
| `fps` | double (e.g. `20`) | Nominal frame rate; the codec adapts the effective rate to the link. |
| `profile` | `extra`, `low`, `medium` | Working resolution / quality profile. `medium` carries both audio and colour. |
| `xdel` | int (e.g. `20`) | Frame-change threshold to trigger a full patch refresh (intra). Lower = better accuracy, more bandwidth. |
| `ydel` | int (e.g. `3`) | Threshold marking a patch as “unchanged” (skip). Lower = smoother motion, more bandwidth. |
| `fec` | `on` \| `off` | Forward Error Correction. Recommended **off** (it reduces effective quality in this regime). |
| `mtu` | int (e.g. `1200`) | UDP MTU for packetization. |
| `key` | string | Shared passphrase enabling ChaCha20-Poly1305 encryption for that channel (video command or audio command — set independently, same value on both ends of a given channel). Omit for an unencrypted stream. |
| `audio` | `on` \| `off` | *(video command only)* Embed audio directly in the picture stream instead of using the standalone audio channel. |
| `audio_device` | string | Capture device name. |
| `audio_sr` | int (e.g. `48000`) | Audio sample rate. |

### Colour parameters (defined as constants in the source)

| Constant | Purpose |
| :--- | :--- |
| `CHROMA_RATIO` | Chroma plane side as a fraction of the luma patch (≈1/6). Lower = less colour, more bandwidth headroom. |
| `CHROMA_GAIN` | Saturation boost to counter subsampling desaturation. `1.0` off; ~`1.3` recommended. |
| `CHROMA_SKIP_MSE` | How much a region must change before its colour is re-sent. Higher = colour updates less often (saves bandwidth). |

---

## 🎓 Citation

If you use this research, the codec concept, or the compiled binaries in an academic or professional context, please cite the paper:

> Rufo, D. (2026). Continuous Keyframe-Free Compression for Real-Time Audio-Video Streaming over Ultra-Low Bandwidth 2G Networks (Version 1.0.0). Zenodo. [https://doi.org/10.5281/zenodo.20406856](https://doi.org/10.5281/zenodo.20406856)

---

## 🤝 Commercial Inquiries & Collaborations

This work is part of a potential patent disclosure. For commercial licensing, production-grade implementations, or research collaborations, contact:

**xdaniele.rufox@gmail.com**

---

## Support my Research 🚀

If you find this project useful for your benchmarks or academic evaluation, consider supporting this independent research:

[![Donate with PayPal](https://img.shields.io/badge/Donate-PayPal-blue.svg)](https://paypal.me/xdanielex272)
[![Donate with BTC](https://img.shields.io/badge/Donate-Bitcoin-orange.svg)](#)
[![Donate with USDT](https://img.shields.io/badge/Donate-Tether-green.svg)](#)

* **Bitcoin (BTC):** `bc1q4l9v8welwr6mp4g6uc2t7ex0n274malynq6yqj`
* **Tether (USDT - TRC20):** `TA3m7pqk1mTgZtFQHf7KufAqnaqsN95kPh`

---

*© 2026 - Continuous Keyframe-Free Compression*
