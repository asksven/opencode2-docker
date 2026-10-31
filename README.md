# OpenCode 2 Docker

This repository provides a Docker image for running [OpenCode 2](https://github.com/anomalyco/opencode) as a
non-root, socket-capable container with the toolchain a coding agent needs. It is the sibling of
`asksven/opencode-docker` (opencode 1.x): this image tracks the **2.x** line and installs the `@opencode/cli`
npm package.

The image is automatically built and updated via GitHub Actions + [Renovate](https://renovatebot.com). Renovate
watches the npm `@opencode/cli` package, opens a PR bumping the pinned `OPENCODE_VERSION` in the `Dockerfile`, and
on merge the `update-image` workflow rebuilds and republishes.

Published images are available on GitHub Container Registry at `ghcr.io/asksven/opencode2-docker`, tagged with the
corresponding OpenCode version (e.g. `ghcr.io/asksven/opencode2-docker:2.2.5`) and `latest`.

> Why a separate image instead of the official `ghcr.io/anomalyco/opencode`? The upstream image is a minimal
> Alpine (musl) image containing only the `opencode` binary + ripgrep. This image keeps a Debian/glibc base with
> the full agent toolchain (git, ssh, Docker CLI + compose + buildx, gh, glab, hadolint, shellcheck, uv/Python +
> ruff, ripgrep, fd, jq) and adds OpenCode 2 on top, matching the opencode 1.x image's environment.

## Prerequisites

- Docker
- Docker Compose

## Usage

You can run the OpenCode server using the provided `docker-compose.yml`:

```bash
docker compose up -d
```

This will start the OpenCode server on `http://localhost:4096`.

## Configuration

### Environment Variables (set in .env)

- `TZ` - Timezone setting
- `OPENCODE_PASSWORD` - Server password. In OpenCode 2 the server username is always `opencode` and only the
  password is configurable (V1's `OPENCODE_SERVER_USERNAME`/`OPENCODE_SERVER_PASSWORD` pair is gone).
- `OPENROUTER_API_KEY` - LLM provider key passed through to the container.

Optional:
- `OPENCODE_LOG_LEVEL` - Raise server log verbosity (e.g. `DEBUG`).

### Volumes

For persistent data and configuration, the following volumes can be mounted:

- `./data/share:/home/opencode/.local/share/opencode` - Shared data, including `auth.json` for LLM provider pre-configuration
- `./data/state:/home/opencode/.local/state/opencode`
- `./data/config:/home/opencode/.config/opencode` - Configuration files, including `opencode.json` (config paths are unchanged in V2)

### Docker CLI Access

The image includes the Docker CLI, Compose v2 plugin, and Buildx plugin, but does not include a Docker daemon. To
use the CLI with a host daemon, explicitly mount the host socket:

```bash
docker run --rm \
  -v /var/run/docker.sock:/var/run/docker.sock \
  ghcr.io/asksven/opencode2-docker:latest \
  docker version
```

When the socket is mounted and the container uses the default root-starting entrypoint, the entrypoint grants the
unprivileged process access through the socket's numeric group ID. Overriding the container user bypasses this
setup and may make a `0660` socket inaccessible. Docker socket access is effectively root-level access to the host,
so only enable this mount for trusted workloads.

### Server / web

`opencode serve` runs the server (and its web UI) in the foreground, which is what the image starts by default:

```bash
opencode serve --hostname 0.0.0.0 --port 4096
```

Connect a client from another machine with `opencode --server http://host:4096`. For a one-time web link, run
`opencode pair`. See the [OpenCode 2 docs](https://opencode.ai/v2/docs/) for the full client/server model.

## Building

The Docker image is built for both `linux/amd64` and `linux/arm64` architectures. Images are automatically pushed
to GitHub Container Registry (`ghcr.io/asksven/opencode2-docker`) with version tags matching OpenCode releases and
a `latest` tag for the most recent release.

To build and load a local image, build one platform at a time:

```bash
docker buildx build --load -t opencode2-docker:local .
```

To publish a multi-platform image, authenticate to GitHub Container Registry and use a multi-platform-capable
Buildx builder:

```bash
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  --push \
  -t ghcr.io/asksven/opencode2-docker:<tag> \
  .
```

## License

Released under the [MIT license](LICENSE).

## Disclaimer

This repository is not affiliated with the OpenCode project. It is an independent effort to provide a Docker image
for OpenCode 2 users. The OpenCode project is developed and maintained by the Anomaly team.