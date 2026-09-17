# DDEV #8832 demo: custom build-service base image collisions

This project reproduces [ddev/ddev#8832](https://github.com/ddev/ddev/issues/8832):
when more than one custom compose service in a DDEV project uses `build:`
against the same default base image, DDEV's documented naming convention
(`image: ${BASE_IMAGE:-example:latest}-${DDEV_SITENAME}-built`) makes the
built images collide on one Docker tag, so the second build silently
overwrites the first — containers end up running the wrong image.

It defines three services that all build from `ubuntu:24.04` in the three
ways DDEV supports `build:`, each with a per-service tag qualifier that
avoids the collision (`-svc1-`, `-svc2-`, `-svc3-`):

| Service | Build style | File |
| --- | --- | --- |
| `svc1` | separate `Dockerfile` | [.ddev/svc1/Dockerfile](.ddev/svc1/Dockerfile) |
| `svc2` | separate `Dockerfile` | [.ddev/svc2/Dockerfile](.ddev/svc2/Dockerfile) |
| `svc3` | `dockerfile_inline` (the pattern from the [custom-compose-files docs](https://ddev.readthedocs.io/en/stable/users/extend/custom-compose-files/)) | [.ddev/docker-compose.svc3.yaml](.ddev/docker-compose.svc3.yaml) |

Each container writes a distinct `/marker.txt` at build time, so you can tell
at a glance whether each service is really running its own image or has been
silently overwritten by another service's build.

## 1. Reproduce the tag collision (any DDEV version)

The collision itself is a plain Docker fact, not something a DDEV version
changes: if two services' `image:` tags are literally identical, only one
entry can ever exist in `docker images`, and whichever service built last
"wins" for both containers.

To see it, edit all three `.ddev/docker-compose.svc*.yaml` files and remove
each service's name from the tag, so all three read exactly the same,
matching DDEV's documented convention literally:

```diff
- image: ${BASE_IMAGE:-ubuntu:24.04}-${DDEV_SITENAME}-svc1-built
+ image: ${BASE_IMAGE:-ubuntu:24.04}-${DDEV_SITENAME}-built
```

(do this for `svc1`, `svc2`, and `svc3` — for `svc3` the variable is
`BASE_IMAGE` too, see the file)

Then:

```bash
ddev restart
docker images | grep issue8832-demo   # only ONE ubuntu-based image, not three
docker exec ddev-issue8832-demo-svc1 cat /marker.txt
docker exec ddev-issue8832-demo-svc2 cat /marker.txt
docker exec ddev-issue8832-demo-svc3 cat /marker.txt
# all three print the same marker — whichever service built last
```

Revert the edit (`git checkout .ddev`) before moving on.

## 2. Reproduce the broken pre-pull/describe behavior (DDEV v1.25.4 and earlier)

With the repo's default, already-unique tags (`-svc1-`, `-svc2-`, `-svc3-`),
the collision above doesn't happen — but on DDEV v1.25.4 and earlier this
"obvious" fix breaks a different thing: DDEV finds the image to pre-pull by
string-trimming the `-${DDEV_SITENAME}-built` suffix off the tag, and doesn't
know how to remove a service-name segment too, so it tries to pull the local
tag itself as if it were a real registry reference.

Install DDEV v1.25.4 (or any released version without the #8832 fix), then:

```bash
ddev start
# non-fatal warning: Unable to pull Docker images: ... failed to resolve
# reference "docker.io/library/ubuntu:24.04-issue8832-demo-svc1": not found

ddev debug download-images
# hard failure, same underlying reference
```

The project still runs (the warning is non-fatal at `start`), but offline
pre-caching is broken for these services.

## 3. Confirm the fix

Build a `ddev` binary from the branch/PR that fixes #8832 (resolves the base
image by reading the Dockerfile/`dockerfile_inline` and `build.args` directly,
instead of parsing it out of the tag), put it first on `PATH`, then:

```bash
ddev start
# no pull warning

ddev describe -j | jq -r '.raw.services | to_entries[] | select(.key|test("svc")) | "\(.key): \(.value.image)"'
# svc1: ubuntu:24.04
# svc2: ubuntu:24.04
# svc3: ubuntu:24.04

ddev debug download-images
# ubuntu:24.04 pulled once, for all three services — success
```

## Cleanup

```bash
ddev delete -O
```
