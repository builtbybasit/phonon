# phonon

Local OpenAI-compatible speech-to-text server for [Phonon-2.5 English](https://www.fermionresearch.com/research/phonon-2.5) (English ASR, 158 MB, CC-BY-4.0), via the `fermion-research` package. Upstream docs: [fermionresearch/phonon](https://github.com/fermionresearch/phonon).

- **macOS (Apple Silicon):** GPU via MLX
- **Windows / Linux / Intel Mac:** CPU (x86-64 with SSE4.1+, or 64-bit Arm). `uv run fermion describe phonon-2.5-en` shows the CPU tier used.
- **NVIDIA GPU (Windows / Linux):** CUDA via Docker — see [NVIDIA GPU](#nvidia-gpu)

## Setup

Requires [uv](https://docs.astral.sh/uv/).

```bash
uv sync
```

The right backend is picked per platform (`fermion-research[mlx]` only adds MLX on Apple Silicon). The model downloads on first run to `~/.cache/fermion/`.

## Run

```bash
uv run fermion serve phonon-2.5-en --port 8010
```

First start compiles shaders (~20 s on Mac). Useful flags: `--host 0.0.0.0`, `--api-key KEY`, `--cors`, `--threads N` (CPU), `--unix-socket PATH` (owner-only socket instead of an API key). See `uv run fermion serve -h`.

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
  -F file=@demo/example3.wav -F model=phonon-2.5-en
```

```json
{"text": "Alright, so have you ever heard of a little thing named text to speech? ..."}
```

Word timestamps: add `-F response_format=verbose_json -F 'timestamp_granularities[]=word'`.

Hotwords (up to 25 names/terms to favour): add `-F hotwords="Ada, Quillon"`. See [hotwords](https://github.com/fermionresearch/phonon/blob/main/docs/hotwords.md).

OpenAI SDK:

```python
from openai import OpenAI

client = OpenAI(base_url="http://127.0.0.1:8010/v1", api_key="<same-key-as-.env>")
with open("demo/example3.wav", "rb") as f:
    print(client.audio.transcriptions.create(model="phonon-2.5-en", file=f).text)
```

Endpoints: `POST /v1/audio/transcriptions`, `GET /v1/models`, WebSocket `/v1/audio/stream` for live audio ([protocol](https://github.com/fermionresearch/phonon/blob/main/docs/server.md#get-v1audiostream-websocket)). No chat endpoints (speech model).

## CLI

```bash
uv run fermion transcribe phonon-2.5-en demo/example2.wav          # one file
uv run fermion transcribe phonon-2.5-en demo/example2.wav --json   # with word timestamps
uv run fermion transcribe phonon-2.5-en demo/example2.wav --hotwords "Ada, Quillon"   # favour names/terms
uv run fermion listen phonon-2.5-en                                # live microphone (Apple Silicon)
```

Other models (`phonon-2`, `phonon-2.5` (25 languages), `phonon-1`, `phonon-1-micro`, `phonon-1-big`): `uv run fermion transcribe phonon-1 file.wav`. `uv run fermion models` lists them.

More: [CLI](https://github.com/fermionresearch/phonon/blob/main/docs/cli.md), [server](https://github.com/fermionresearch/phonon/blob/main/docs/server.md), [troubleshooting](https://github.com/fermionresearch/phonon/blob/main/docs/troubleshooting.md).
