---
name: docker
description: Use the host Docker engine from inside the harness to build, run, inspect, or clean up containers and images. Apply when a task needs docker commands (build, run, exec, ps, images, rm, prune, compose, buildx), needs a throwaway container to run tooling or a service, or must reproduce a docker-based workflow. Requires a dsh runtime that ships the docker CLI (with compose and buildx) and mounts the host Docker socket at /var/run/docker.sock.
---

# Docker from inside the harness

The docker CLI talks to the Docker engine of the host machine (for example
Docker Desktop) over a Unix socket that the runtime mounts at
`/var/run/docker.sock`. Containers you start therefore run on the host engine,
not inside the harness sandbox, and can mount host paths.

## Check availability first

Run `docker info` before anything else.

- `docker: command not found` — the runtime image does not ship the docker
  CLI. The image must be rebuilt from a release that includes the docker
  utilities (dsh-v0.1.3-rc.1 or later) — see `docker/README.md`.
- `Cannot connect to the Docker daemon at unix:///var/run/docker.sock` — the
  host engine is off (start Docker Desktop) or the socket is not mounted.
- `permission denied while trying to connect` — the runtime user is not in the
  socket's group; the deployment must map the socket group (see
  `docker/README.md`).

Report what you observe instead of retrying blindly.

## Permissions and sandbox

The docker CLI connects to the daemon over a Unix socket. Under a confining
filesystem sandbox that connect is unreliable, so:

- When the bash tool exposes a `sandbox_permissions` parameter, pass
  `sandbox_permissions=danger-full-access` with a one-line `justification`
  for every docker command.
- When the environment already runs with full access (no
  `sandbox_permissions` parameter in the tool schema), run docker commands
  directly.

Never switch to `DOCKER_HOST=tcp://...` on your own — only if the user has
configured a remote engine.

## Working rules

- Treat the daemon as root on the host machine. Destructive operations
  (`docker system prune -f`, `docker rmi -f`, `docker volume rm`,
  `docker network rm`) only when the user explicitly asks for cleanup.
- Prefer `--rm` for throwaway containers so they do not linger.
- To exchange files, mount a workspace path explicitly:
  `docker run --rm -v "$PWD:/work" -w /work <image> <command>`.
- Containers share the host network namespace only when you pass
  `--network host`. Publishing ports (`-p`) claims host ports — keep them
  explicit and minimal; avoid `-p` for one-shot work.
- Give heavy jobs resource bounds (`--memory`, `--cpus`) and always prefer
  running a command to completion in the foreground over leaving a container
  running.
- Use `docker compose` for multi-container stacks from a compose file and
  `docker buildx build` when the user asks for a specific platform or cache
  export. Both plugins are installed in the runtime image.
- Name images and containers (`-t name:tag`, `--name`) so later steps can
  refer to them deterministically.

## Typical flows

Build and run an image from the current directory:

```bash
docker build -t work:local .
docker run --rm -v "$PWD:/work" -w /work work:local <command>
```

Inspect and clean state:

```bash
docker ps -a
docker images
docker system df
```

Reuse one long-lived helper container across steps:

```bash
docker run -d --name helper -v "$PWD:/work" -w /work alpine tail -f /dev/null
docker exec helper <command>
docker rm -f helper
```
