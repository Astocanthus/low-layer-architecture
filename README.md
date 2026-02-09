# low-layer-architecture

[![License: LOW-LAYER](https://img.shields.io/badge/License-LOW--LAYER-blue.svg)](LICENSE)
[![Release](https://img.shields.io/github/v/release/Astocanthus/low-layer-architecture)](https://github.com/Astocanthus/low-layer-architecture/releases/latest)
[![Issues](https://img.shields.io/github/issues/Astocanthus/low-layer-architecture)](https://github.com/Astocanthus/low-layer-architecture/issues)

Platform architecture documentation for LOW-LAYER infrastructure.

This repository is the single source of truth for all architectural decisions, platform hierarchy, repository structure, and infrastructure design of the LOW-LAYER platform.

---

## Features

- **Platform Hierarchy**: Complete tenant, organisation, and VPC environment architecture with resource modes (mutualized/dedicated), service bastion design, and service catalog.
- **Repository Structure**: Multi-repository organisation covering API, CLI, console, website, Terraform provider, and Helm charts with inter-repository dependencies and versioning strategy.
- **Infrastructure Layers**: Low-level infrastructure design including bare metal bootstrap, root Kubernetes cluster, Fedora CoreOS images, OpenBao PKI hierarchy, operator/agent model, and recursive deployment architecture.
- **Console Architecture**: Core module design (authentication, HTTP interceptors, ApiService, BaseRepository), shared Zod schemas, feature modules (dashboard, organizations, catalog), MSW mock system.
- **Graph Editor**: Rete.js v2 visual topology editor with typed socket/pin system, 5-layer connection validation, relation editor (link blueprints), auto-chain, sub-graph editor, and OCI image sub-resources.

---

## Project Structure

```
low-layer-architecture/
├── docs/
│   ├── platform.md              # Platform hierarchy and service catalog
│   ├── infrastructure.md        # Bare metal layers, operator, agent
│   ├── repositories.md          # Multi-repository overview and CI/CD
│   └── console/
│       ├── architecture.md      # Console core modules, auth, features
│       └── graph-editor.md      # Rete.js graph editor architecture
├── CHANGELOG.md
├── CLAUDE.md
├── LICENSE
└── README.md
```

---

## License

This project is licensed under the [LOW-LAYER Source Available License](LICENSE).
- **Free** for up to 5 users
- **Commercial license** required for larger teams
- Contact: contact@low-layer.com

---

## Author

**LOW-LAYER Engineering**

- Website: [low-layer.com](https://low-layer.com)
- Contact: contact@low-layer.com

---

## Acknowledgments

- Built for [LOW-LAYER](https://low-layer.com) platform infrastructure
