---
name: Feature request
about: Suggest an idea for this project
title: '[FEATURE] Containerize application for Kubernetes deployment'
labels: enhancement
type: Feature
assignees: ''
---

## Summary

Package the OS application as a container image so it can be deployed onto container orchestration platforms like Kubernetes. This will enable consistent, scalable, and portable deployments across environments (development, staging, production).

### Acceptance Criteria

- [ ] A `Dockerfile` is added to the repository root that produces a minimal, production-ready container image using a multi-stage build (builder stage compiles the Bun application, runtime stage uses a distroless or slim base image)
- [ ] A `.dockerignore` file is added to exclude unnecessary files (node_modules, .git, .infer, local configs) from the build context
- [ ] The container image is published to GitHub Container Registry (ghcr.io) as part of the release workflow (`.github/workflows/release.yml`)
- [ ] A `docker-compose.yml` example is provided for local development and testing
- [ ] A `deployment.yaml` Kubernetes manifest example is provided demonstrating:
  - Deployment with resource requests/limits
  - Service configuration
  - ConfigMap for environment-specific settings
  - Horizontal Pod Autoscaler (HPA) for scaling
- [ ] Documentation is added to `README.md` covering:
  - Building the image locally
  - Running with Docker Compose
  - Deploying to a Kubernetes cluster (minikube, kind, or production cluster)
  - Configuration via environment variables
