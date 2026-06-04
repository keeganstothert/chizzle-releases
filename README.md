# chizzle-releases

Public **release metadata** for [chizzle](https://chizzle.example.com) — tags,
GitHub Releases, and the upgrade runbook ([`UPGRADING.md`](UPGRADING.md)).

The chizzle **source** is private. This repo exists so a self-hosted
chizzle-server can check *"is there a newer release?"* with an anonymous,
public API call (`/repos/keeganstothert/chizzle-releases/releases/latest`) — no
token, no access to the source.

It carries **no source code**. The release artifacts are signed container
images on GHCR (`ghcr.io/keeganstothert/chizzle`), pulled on the deployment host
with `docker login`; this repo only records *which version is latest* and how to
upgrade. Verify image signatures against [`cosign.pub`](cosign.pub).
