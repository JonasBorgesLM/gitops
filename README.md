# gitops

**A minimal, end-to-end GitOps pipeline**: push Go code, and a GitHub Actions
workflow builds the image, updates the Kubernetes manifests with the new tag,
and commits that change back to this repository.

The application is a hello-world HTTP server on purpose. The subject here is the
delivery mechanism.

## The idea

In a push-based pipeline, CI builds an image and then reaches into the cluster
with `kubectl apply`. That means CI holds cluster credentials, and the cluster's
actual state lives nowhere you can read.

GitOps inverts it. The repository holds the desired state as manifests, and
deployment is a **commit** — a change to `k8s/kustomization.yaml` pinning a new
image tag. What is running becomes reviewable, diffable and revertable by
`git revert`, and a reconciler (Argo CD, Flux) pulls it into the cluster instead
of CI pushing.

This repository implements the left half — the part that turns a code push into
a manifest commit. A reconciler watching `k8s/` would complete the loop.

## The pipeline

`.github/workflows/cd.yaml`, on every push to `master`:

1. **Build and push** the image to Docker Hub, tagged with both the commit SHA
   and `latest`.
2. **Set up Kustomize.**
3. **Update the manifest** — `kustomize edit set image goserver=<user>/gitops:$GITHUB_SHA`
   rewrites the `images:` block in `k8s/kustomization.yaml`.
4. **Commit and push** that edit back to the repository.

Step 4 is the one that matters: after it, `k8s/kustomization.yaml` names an
immutable, content-addressed tag. The commit SHA is the deployed version, so
"what is in production" is answerable by reading the file.

### Why the SHA tag, not `latest`

`latest` is a moving pointer. Two clusters pulling `latest` an hour apart can
run different code while both claim to be up to date, and a rollback has nothing
to roll back to. The SHA tag makes the deployed artifact identifiable and the
rollback a revert.

## Layout

```
main.go                       # the HTTP server — port 8080, one handler
Dockerfile                    # multi-stage: golang builder → scratch
k8s/
├── deployment.yaml           # references the logical image name `goserver`
├── service.yaml
└── kustomization.yaml        # maps `goserver` to a real image:tag
.github/workflows/cd.yaml     # the pipeline
```

The indirection in `kustomization.yaml` is what makes this work: `deployment.yaml`
never changes, because it refers to `goserver`, and only the Kustomize image
mapping is rewritten per deploy.

The image is built `CGO_ENABLED=0` and shipped `FROM scratch` — a container with
a single static binary and nothing else: no shell, no package manager, and
nearly no attack surface.

## Setup

The workflow needs two repository secrets:

| Secret | Purpose |
| --- | --- |
| `DOCKER_USERNAME` | Docker Hub user; also used as the image namespace |
| `DOCKER_PASSWORD` | Docker Hub access token (prefer a token over a password) |

`GITHUB_TOKEN` is provided automatically, and the workflow declares
`permissions: contents: write` so it can push the manifest commit.

## Running locally

```bash
go run main.go        # http://localhost:8080
kubectl apply -k k8s/
```

## Scope and known rough edges

A study project. Things a real pipeline would want:

- **No reconciler.** Nothing pulls `k8s/` into a cluster yet — Argo CD or Flux
  is the missing half.
- **The commit step has no guard.** `git commit -am` fails the job when the tag
  did not change, since there is nothing to commit.
- **Pinned to `master`,** with no staging environment or promotion step.
- **Dated actions.** `actions/checkout@v2` and `docker/build-push-action@v1.1.0`
  are several majors behind.

## Related

[`k8s`](https://github.com/JonasBorgesLM/k8s) is the step before this one: the
same kind of server, with the manifests applied by hand.
