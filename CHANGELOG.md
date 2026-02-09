# Changelog

All notable changes to the low-layer-architecture project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

---

## [Unreleased]

### Planned

- Additional architecture documents as platform evolves

---

## [0.1.0] - 2026-02-09

### Added

- **Platform Infrastructure Architecture** (`platform.md`)
  - Tenant, organisation, and VPC environment hierarchy
  - Service bastion design (Keycloak, OpenBao, K8s control plane)
  - Resource modes: mutualized (cost-saver) and dedicated (performance)
  - Service catalog with identity, persistence, messaging, observability, storage, and AI/ML categories
  - Key architecture principles (tenant isolation, flexible allocation, no lock-in)

- **Multi-Repository Structure** (`repositories.md`)
  - Repository overview: meta, API, CLI, website, console, Terraform provider, Helm charts
  - Detailed structure and tech stack for each repository
  - Inter-repository dependency graph
  - Shared components: API client SDK, Proto/OpenAPI specs, legal pages
  - Versioning strategy, development workflow, and CI/CD pipeline

- **Low-Level Infrastructure Architecture** (`infrastructure.md`)
  - 5-layer model: bare metal, root cluster, platform services, OpenStack, org clusters
  - Fedora CoreOS immutable images (master/worker) with boot sequence
  - OpenBao PKI hierarchy with per-organisation intermediate CAs
  - Operator (`ll-operator`) and agent (`ll-agent`) design
  - Organisation types: standard (OpenStack VMs) and bare metal
  - Recursive deployment model
  - Rook Ceph unified storage architecture
  - Product tiers: Free (native), Paid (platform), Enterprise (self-hosted)

---

[0.1.0]: https://github.com/Astocanthus/low-layer-architecture/releases/tag/v0.1.0
