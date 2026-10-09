# phonon

Local OpenAI-compatible speech-to-text server for [Phonon-2](https://www.fermionresearch.com/research/phonon-2/) (English ASR, 164 MB, CC-BY-4.0), via the `fermion-research` package.

- **macOS (Apple Silicon):** GPU via MLX
- **Windows / Linux (x64):** CPU via torch
- **NVIDIA GPU (Windows / Linux):** CUDA via Docker — see [NVIDIA GPU](#nvidia-gpu)

## Setup

Requires [uv](https://docs.astral.sh/uv/).

```bash
uv sync
```

The right backend is picked per platform. The model downloads on first run to `~/.cache/fermion/`.

## Run

```bash
uv run phonon serve --port 8010
```

First start compiles shaders (~20 s on Mac). Useful flags: `--host 0.0.0.0`, `--api-key KEY`, `--cors`, `--threads N` (CPU). See `uv run phonon serve -h`.

## NVIDIA GPU

The pip package has no CUDA engine (installing CUDA torch does nothing — the CPU engine never uses it). Fermion ships CUDA only as a Docker image, which serves the same API on the same port:

Create a `.env` file with an API key before starting the container:

```dotenv
API_KEY=<strong-random-key>
```

```bash
docker compose up -d
```

Compose passes `API_KEY` to the container as `PHONON_API_KEY`. The container binds to `0.0.0.0`, while Docker publishes it only on `127.0.0.1`.

- **Windows:** [Docker Desktop](https://docs.docker.com/desktop/setup/install/windows-install/) with the WSL2 backend and a current NVIDIA driver. No CUDA toolkit needed.
- **Linux:** Docker + [NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html).

Check the GPU is visible: `docker run --rm --gpus all ubuntu nvidia-smi`. Logs: `docker compose logs -f`.

## Use

```bash
curl http://127.0.0.1:8010/v1/audio/transcriptions \
  -H "Authorization: Bearer <same-key-as-.env>" \
  -F file=@demo/example3.wav -F model=phonon-2
```

```json
{"text": "All right, so have you ever heard of a little thing named text to speech? ..."}
```

Word timestamps: add `-F response_format=verbose_json -F 'timestamp_granularities[]=word'`.

OpenAI SDK:

```python
from openai import OpenAI

client = OpenAI(base_url="http://127.0.0.1:8010/v1", api_key="<same-key-as-.env>")
with open("demo/example3.wav", "rb") as f:
    print(client.audio.transcriptions.create(model="phonon-2", file=f).text)
```

Endpoints: `POST /v1/audio/transcriptions`, `GET /v1/models`. No chat endpoints (speech model).

## CLI

```bash
uv run phonon transcribe demo/example2.wav          # one file
uv run phonon transcribe demo/example2.wav --json   # with word timestamps
uv run phonon listen                                # live microphone
```
