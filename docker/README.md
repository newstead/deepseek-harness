# Docker utilities

This folder carries the docker capability that a release of this repository
ships to its runtimes. DeepSeek Harness itself does not hardcode docker:
running docker from the harness is a deployment property, because the agent's
`bash` tool executes in the same container as the harness process. A runtime
can use docker when three conditions hold:

1. the runtime image contains the docker CLI plus the `docker compose` and
   `docker buildx` plugins;
2. the host Docker socket is mounted into the runtime at
   `/var/run/docker.sock` (the host engine is Docker Desktop on macOS in the
   reference deployment);
3. the harness process user can reach the socket (the deployment maps the
   socket's group onto the runtime user).

The sandbox vocabulary of the harness covers filesystem effects only: there is
no code-level allowlist that blocks docker, sockets, or network. Under a
confining filesystem sandbox, socket access is unreliable, so the skill tells
the agent to request `danger-full-access` for docker commands.

## Contents

- `SKILL.md` — the `docker` skill. Deployments install it into the harness
  user-skill root `$DSH_HOME/skills/docker/SKILL.md` (the skill loader reads
  `$DSH_HOME/skills`), so agents in the web UI and CLI see it through the
  `skill` tool. The reference deployment's entrypoints copy it from the
  built image on every container start, which keeps a rebuilt image and its
  skills in lockstep.
- `README.md` — this file.

## Released since

The docker skill entered the repository with the fork release `dsh-v0.1.3-rc.1`.
Runtimes built from that tag or later and wired per the conditions above can
run docker from the harness; earlier tags cannot.

## Verify inside a runtime

Ask the agent to run `docker info`, or check from the harness CLI container:

```bash
docker version --format '{{.Server.Version}}'
docker compose version
docker buildx version
```
