---
name: ffmpeg-mastery
description: Comprehensive FFmpeg guide for media processing, conversion, filtering, and streaming. Covers CLI usage for video, audio, and image manipulation.
---

# FFmpeg Mastery

## Overview

FFmpeg is the "Swiss Army Knife" of media processing. It acts as the core engine for transcoding, streaming, and filtering multimedia content. This skill provides production-ready commands and patterns for manipulating video, audio, and images.

## Core Concepts

- **Container**: The file format (e.g., `.mp4`, `.mkv`, `.mov`) that holds the streams.
- **Stream**: The actual data (Video, Audio, Subtitle) inside the container.
- **Codec**: The algorithm used to compress/decompress the stream (e.g., H.264, AAC, VP9).
- **Filter**: A process applied to the raw stream (e.g., scaling, overlay, color correction).

## Basic Syntax

```bash
ffmpeg -i [input_file] [options] [output_file]
```

## Common Operations

### Video Conversion

**Convert to MP4 (H.264/AAC):**

```bash
ffmpeg -i input.mov -c:v libx264 -preset slow -crf 22 -c:a aac -b:a 128k output.mp4
```

- `-c:v libx264`: Use H.264 video codec.
- `-preset slow`: Better compression efficiency (choices: ultrafast, superfast, veryfast, faster, fast, medium, slow, slower, veryslow).
- `-crf 22`: Constant Rate Factor (18-28 is good range, lower is better quality).
- `-c:a aac`: Use AAC audio codec.

**Extract Audio:**

```bash
ffmpeg -i video.mp4 -vn -c:a mp3 -b:a 192k audio.mp3
```

**Mute Video:**

```bash
ffmpeg -i video.mp4 -an -c:v copy silent.mp4
```

### Image Operations

**Video to Images (FPS = 1):**

```bash
ffmpeg -i video.mp4 -vf fps=1 out%d.png
```

**Images to Video:**

```bash
ffmpeg -framerate 24 -i img%03d.png -c:v libx264 -pix_fmt yuv420p output.mp4
```

## Advanced Filters

### Scaling & Cropping

**Scale to 1080p (maintaining aspect ratio):**

```bash
ffmpeg -i input.mp4 -vf "scale=-1:1080" output.mp4
```

**Crop Center:**

```bash
ffmpeg -i input.mp4 -vf "crop=w=100:h=100:x=(in_w-100)/2:y=(in_h-100)/2" output.mp4
```

### Watermark Overlay

**Overlay transparent PNG (bottom-right):**

```bash
ffmpeg -i main.mp4 -i logo.png -filter_complex "[1]scale=100:-1[logo];[0][logo]overlay=W-w-10:H-h-10" output.mp4
```

### Concatenation

**Join list of files (files.txt):**

```text
file 'part1.mp4'
file 'part2.mp4'
```

```bash
ffmpeg -f concat -safe 0 -i files.txt -c copy output.mp4
```

## Performance Tuning

### Hardware Acceleration (NVIDIA NVENC)

Use `-c:v h264_nvenc` or `-c:v hevc_nvenc` for GPU encoding.

```bash
ffmpeg -i input.mp4 -c:v h264_nvenc -preset p4 -cq 20 output.mp4
```

### Two-Pass Encoding (Target Bitrate)

Useful for streaming where exact bitrate is required.

```bash
# Pass 1
ffmpeg -y -i input.mp4 -c:v libx264 -b:v 2000k -pass 1 -an -f null /dev/null

# Pass 2
ffmpeg -i input.mp4 -c:v libx264 -b:v 2000k -pass 2 -c:a aac -b:a 128k output.mp4
```

## Reference

- [FFmpeg Documentation](https://ffmpeg.org/ffmpeg.html)
- [Filters Documentation](https://ffmpeg.org/ffmpeg-filters.html)
- [Codecs Documentation](https://ffmpeg.org/ffmpeg-codecs.html)
