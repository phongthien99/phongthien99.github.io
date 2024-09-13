---
title: Building an HTTP Server for Multi-Platform Data Storage
author: phongthien99
date: 2024-06-24 00:40:00 +0800
categories: [http-server]
tags: [idea]
math: true
media_subpath: '/posts/20180809'
---

# Building an HTTP Server for Multi-Platform Data Storage

## Introduction

In the deployment of Video On Demand (VOD) services, transcoding and storing video require a flexible and efficient system. To address this issue, we can build an HTTP server to manage the writing and retrieving of data from FFmpeg to various storage platforms such as S3, FTP, OS, and HTTP.

## Solution

We will design an HTTP server with two main endpoints:

1. **PUT /origin/{config}/{filepath}**
    - Receive data from FFmpeg and save it to the configured storage platform.
    - `config` is encoded in base64, decoded to determine the storage platform, and then save the data accordingly (S3, FTP, OS, HTTP). The request can store data on multiple platforms simultaneously.
2. **GET /origin/{config}/{filepath}**
    - Retrieve data from the configured storage platforms based on `config`.

Configure FFmpeg to use the HTTP server for testing:

```bash
ffmpeg -i video.mp4 \
       -c:a aac \
       -c:v h264 \
       -f hls \
       -force_key_frames expr:gte(t,n_forced*2) \
       -hls_base_url /origin/BASE64_ENCODED_CONFIG/segments/ \
       -hls_playlist_type vod \
       -hls_segment_filename http://localhost:8086/origin/BASE64_ENCODED_CONFIG/segments/segment_%03d.ts \
       -hls_time 2 \
       -vf scale=1280:-1 \
       http://localhost:8086/origin/BASE64_ENCODED_CONFIG/manifest/master.m3u8 -y

```

## Conclusion

Building an HTTP server with the structure `/origin/:config/:filePath` helps manage and retrieve data efficiently and simplifies the storage process in a VOD system. This approach minimizes the complexity in data transmission and configuration handling, meeting the needs of a modern VOD system effectively.