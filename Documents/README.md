# 📚 StreamingApp Documentation Hub

Welcome to the central documentation repository for the **StreamingApp** project. This directory contains all architectural diagrams, deployment guides, troubleshooting manuals, and user tutorials necessary to understand, run, and scale the application.

Below is an index of all available documents:

## 🚀 Application Usage & Visuals
* **[`application guide.md`](./application%20guide.md)**
  The complete end-to-end user tutorial. It covers how a new user can register an account, systematically escalate their privileges to "Administrator" at the database level, upload new videos, and stream content using the platform.
* **[`application screenshots.md`](./application%20screenshots.md)**
  A massive visual reference guide containing high-resolution screenshots of the entire application across all major user flows (Sign Up, Video Player, Admin Dashboard, AWS S3 Buckets, etc.).

## 🚢 CI/CD & Infrastructure
* **[`ci-cd-architecture.md`](./ci-cd-architecture.md)**
  A deep-dive into the modern Continuous Integration and Continuous Deployment pipeline. Includes sequence diagrams explaining how Github, Jenkins, Amazon ECR, and Kubernetes (EKS) work together via a fully automated Helm GitOps flow.
* **[`helm-deployment.md`](./helm-deployment.md)**
  The absolute master guide on the project's Helm integration. Breaks down the state-of-the-art `.yml` templating logic, `secretKeyRef` security patterns, and the `initContainer` React Nginx hot-patching architecture used to solve Single Page Application (SPA) multi-domain caching issues.
* **[`k8s manifest document.md`](./k8s%20manifest%20document.md)**
  A historical reference detailing the raw native Kubernetes manifests that powered the system before the migration to the dynamic Helm package manager.

## 💻 Commands & Local Setup
* **[`cmd.md`](./cmd.md)**
  The ultimate developer cheat sheet. It contains dozens of meticulously documented commands for AWS authentication, advanced Docker image pushing, deep Kubernetes introspection, ECR login pipelines, and MongoDB terminal patching. If you need a command, it is in here.
* **[`command.md`](./command.md)**
  A legacy, quick-reference command scratchpad (mostly superseded by `cmd.md`).
* **[`installation.md`](./installation.md)**
  The zero-to-hero local machine setup guide. Covers prerequisite software installation and the exact chronological sequence to launch the backend microservices and React frontend.

## 🛠️ Debugging
* **[`troubleshooting.md`](./troubleshooting.md)**
  The official incident and debug log. If the application crashes, pods get stuck in `<Pending>`, AWS ECR denies authorization (`ErrImagePull`), or React gets trapped in Cross-Domain CORS hell, you will find the exact `kubectl` verification commands and proven resolutions here.
