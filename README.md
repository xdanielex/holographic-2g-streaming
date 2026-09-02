# Continuous Keyframe-Free Compression for Real-Time Streaming

A C++ codec for real-time audio-video streaming engineered to deliver a **stable, smooth feed over genuine ultra-low-bandwidth 2G / EDGE networks** and high-cost satellite uplinks — links with high latency and heavy packet loss where conventional predictive codecs fail.

[Download the full tech paper (PDF)](STREAMING2g.pdf)

---

## ✨ What's new in this version

* **Genuine colour.** The earlier release operated close to monochrome. This version introduces **real, natural-colour video** without disturbing the low-bandwidth design: chrominance is carried as a low-resolution plane that is sent only when it actually changes, so a static background keeps its colour at almost no cost. On the **medium** profile it delivers true-colour audio-video that stays fluid on a real 2G/EDGE link.
* **True-to-source implementation.** The README now documents the algorithm actually implemented (see *Core Innovation*) rather than an abstract model.

---

## 📋 System Prerequisites

* **Operating System:** Windows 11 (64-bit) or Windows 10 (64-bit). *(The codec logic is platform-neutral; the streaming shell is Windows-centric.)*
* **[OpenCV](https://opencv.org/)** — C++ headers and a prebuilt library (`opencv_world4xx.lib`) for MSVC.
* **[zstd](https://facebook.github.io/zstd/)** — the entropy-compression library.
* **FFmpeg / ffplay** — used for capture (DirectShow/Pulse), audio (Opus/OGG), and publishing. Must be added to your **PATH**.
* **[MediaMTX](https://github.com/bluenviron/mediamtx/releases)** *(optional)* — the low-latency RTSP/WebRTC relay. Place `mediamtx.exe` in the same folder as the executable; the receiver auto-launches it.

---

## 📺 Live Video Demonstration

[![Watch the 2G Streaming Demo](https://img.youtube.com/vi/2jCk43ZhJ_0/maxresdefault.jpg)](https://www.youtube.com/watch?v=2jCk43ZhJ_0)

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
6. **Transport** — UDP packetization with fragmentation and a checksum, reassembled on the receiver.
7. **Delivery** — RTSP / WebRTC relay via MediaMTX.

---

## 💻 Quick Start & Usage

The framework uses a unified CLI binary for sending and receiving video and audio streams over UDP.

### 1. Video Receiver

```bat
mio_codec_cli_opencv_full_color.exe stream-recv-webcam-full udp://0.0.0.0:5000 out_webdir=web "opts=http_port=8080,mtu=1200,profile=medium,xdel=20,ydel=3,fec=off"
```

### 2. Video Sender

```bat
mio_codec_cli_opencv_full_color.exe stream-send-webcam-full udp://1.2.3.4:5000 "opts=w=320,h=240,fps=20,mtu=1200,profile=medium,dur=0,audio=on,audio_device=Gruppo microfoni (Realtek(R) Audio),audio_sr=48000,xdel=20,ydel=3,fec=off"
```

> `audio=on` embeds the captured audio (Opus/OGG) directly in the picture stream, so the receiver plays it alongside the video. If `audio=off`, video only.

### 3. Standalone Audio Receiver (RTP)

```bat
mio_codec_cli_opencv_full_color.exe stream-recv-audio udp://0.0.0.0:5001
```

### 4. Standalone Audio Sender (RTP)

```bat
mio_codec_cli_opencv_full_color.exe stream-send-audio udp://1.2.3.4:5001 "Gruppo microfoni (Realtek(R) Audio)"
```

*Note: for single-machine local testing you can loop the traffic back, or use an external UDP port-forwarding/remap utility such as portmap.io.*

---

## 🔧 Build

### Windows (MSVC)

Open an **x64 Native Tools Command Prompt for VS** and run (adjust the OpenCV path if yours differs):

```bat
cl /EHsc /std:c++17 mio_codec_cli_opencv_full_color.cpp ^
   /I "C:\opencv\build\include" ^
   /link /LIBPATH:"C:\opencv\build\x64\vc15\lib" opencv_world455.lib zstd.lib Ws2_32.lib ^
   /Fe:mio_codec_cli_opencv_full_color.exe
```

### Linux (g++)

A prebuilt Linux binary is included in the repo (`mio_codec_cli_opencv_full_color`). To rebuild it from source:

```bash
sudo apt-get install -y libopencv-dev libzstd-dev
g++ -std=c++17 -O2 mio_codec_cli_opencv_full_color.cpp -o mio_codec_cli_opencv_full_color \
    $(pkg-config --cflags opencv4) $(pkg-config --libs opencv4) -lzstd -lpthread
```

> **Runtime dependencies (Linux):** the binary links against `libopencv_core/imgproc/videoio/imgcodecs` (OpenCV 4.x) and `libzstd`. `ffmpeg`/`ffplay` are still needed in `PATH` for the audio-capture and decode/webcam pipelines. The source is portable across both platforms; the streaming shell differs (`Winsock` on Windows, POSIX sockets on Linux).

> **Important:** sender and receiver must both run the same binary / format. The colour version writes an extended block header, so a stream encoded with this build must be decoded by this build.

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
| `audio` | `on` \| `off` | Embed audio in the picture stream. |
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
