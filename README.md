<p align="center">
  <img src="assets/logo.png" width="120" alt="coco-release-spec logo" />
</p>

<h1 align="center">coco-release-spec</h1>

<p align="center">
  Dev-facing release engineering specification for Coco projects.
</p>

---

## Overview

A knowledge-only zylos component that packages the Dev-agent portions of the 《开发与运维协同规范》v0.7. It defines how Dev agents should prepare releases, write design documents, maintain config schemas, and produce deployment scripts.

## Install

```bash
zylos add coco-release-spec
```

## Features

- **Design document requirements** — deployment plan and upgrade plan chapters that must accompany every design doc
- **Release checklist template** — structured tables for env var changes, DB migrations, build params, and deploy order
- **Config schema specification** — JSON Schema-based format for declaring all configuration items, types, and constraints
- **Deploy script standards** — interface contracts for `deploy.sh`, `upgrade.sh`, `rollback.sh`, and `healthcheck.sh`
- **Release flow** — Git tag, CI pipeline, and `release-manifest.json` format
- **Application coding standards** — config centralization, naming conventions, and health endpoints

## Usage

This component is loaded automatically when Claude needs guidance on release engineering tasks. The SKILL.md file contains the full specification.

## License

[MIT](LICENSE)
