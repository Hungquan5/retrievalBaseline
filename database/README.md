# Video Keyframe Extraction & Embedding Indexing

This repository provides a pipeline for:

1. **Extracting visual keyframes** from videos using [OpenCLIP](https://github.com/mlfoundations/open_clip) similarity filtering.
2. **Indexing extracted keyframes into [Milvus](https://milvus.io/)** for efficient similarity search and retrieval.

---

## 1. Keyframe Extraction

**Script:** `extract_keyframes.py`

This script processes a folder of videos, extracts representative keyframes based on cosine similarity of OpenCLIP embeddings, and saves:

* Keyframes as **`.webp` images**
* Frame-to-time mappings as **CSV files**

### Usage

```bash
python extract_keyframes.py \
  --input-folder /path/to/videos \
  --output-base ./output-keyframes \
  --clip-threshold 0.93 \
  --skip-frames 5
```

### Key Arguments

* `--input-folder`: Root folder containing videos (`.mp4` by default).
* `--output-base`: Where keyframes and maps will be stored.
* `--clip-threshold`: Cosine similarity threshold (lower similarity → new keyframe).
* `--skip-frames`: Process every `(skip_frames + 1)`th frame.
* `--pattern`: Glob pattern for video files (default: `*.mp4`).

Each processed video produces:

```
output-keyframes/
  ├── maps/
  │   └── video1_map.csv
  ├── video1/
  │   └── keyframes/
  │       ├── keyframe_150.webp
  │       ├── keyframe_300.webp
  │       └── ...
  └── video2/
      └── keyframes/...
```

---

## 2. Milvus Indexing

**Script:** `milvus_index_keyframes.py`

This script takes the extracted keyframes (`.webp`) and inserts their embeddings into a Milvus collection for fast vector similarity search.

### Features

* Multi-GPU or CPU-only encoding
* Batch-wise insertion with configurable flush interval
* Automatic collection creation with HNSW index
* Option to rebuild index after ingestion for faster queries

### Usage

```bash
python milvus_index_keyframes.py \
  --root ./output-keyframes \
  --collection-name AIC25_fullbatch1 \
  --recreate \
  --build-index
```

### Key Arguments

* `--root`: Root folder containing keyframes.
* `--collection-name`: Milvus collection name.
* `--recreate`: Drop and recreate the collection if it exists.
* `--build-index`: Build HNSW index after insertions.
* `--batch-size`: Number of keyframes per encoding batch (default: 16).
* `--flush-interval`: Flush every N inserts (default: 2000).
* `--workers`: `auto` = use all GPUs, `cpu` = single worker on CPU, or integer for #workers.

---

## 3. Database: Milvus

[Milvus](https://milvus.io/) is a high-performance vector database designed for similarity search.
In this project, it stores embeddings for keyframe retrieval.

* We use **HNSW (Hierarchical Navigable Small World Graph)** indexing with **cosine similarity**.
* Schema:

  * `id`: Primary key (INT64)
  * `filepath`: Path to keyframe image
  * `embedding`: 1024-dim float vector (from OpenCLIP)
  * `video_id`: Parent video folder
  * `frame_id`: Frame index in original video

For advanced Milvus features (e.g., partitioning, hybrid search, scaling, query optimization), see the official docs:
👉 [Milvus Documentation](https://milvus.io/docs)

---

## Pipeline Overview

```mermaid
graph TD
    A[Video Files] -->|extract_keyframes.py| B[Keyframe Images + Map CSVs]
    B -->|milvus_index_keyframes.py| C[Milvus Collection]
    C --> D[Vector Search / Retrieval]
```

---

## Requirements

* Python 3.9+
* CUDA-enabled GPU(s) recommended for speed
* Libraries: `torch`, `open_clip_torch`, `tqdm`, `pymilvus`, `PIL`, `opencv-python`

Install dependencies:

```bash
pip install torch torchvision tqdm opencv-python pillow pymilvus open_clip_torch
```

---

✅ With this pipeline:

* You can extract **compact sets of representative frames** from large video datasets.
* Store them in **Milvus** for fast semantic search and retrieval.

