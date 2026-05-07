# Custom runner image for the fifa-runners scale set.
#
# Base image is the official ARC runner. We layer on:
#   - kubectl              (CD steps: kubectl set image, kubectl rollout status, kubectl apply -k)
#   - helm                 (third-party chart upgrades from CI when needed)
#   - docker CLI           (handy for buildctl context, image inspection)
#   - buildctl             (BuildKit client. Talks to the in-cluster rootless BuildKit Deployment)
#   - jq, yq, curl, gettext (workflow plumbing)
#   - Node.js 20 + npm     (runs Vitest, npm scripts, lint for FIFA_API_APP)
#
# We do NOT install a Docker daemon. Builds happen in BuildKit, jobs talk to it over gRPC.

ARG ARC_RUNNER_VERSION=2.334.0
FROM ghcr.io/actions/actions-runner:${ARC_RUNNER_VERSION}

USER root

ARG TARGETARCH=amd64
ARG KUBECTL_VERSION=v1.33.6
ARG HELM_VERSION=v3.16.4
ARG BUILDKIT_VERSION=v0.18.2
ARG NODE_MAJOR=20
ARG YQ_VERSION=v4.44.5

# OS deps. Keep this list minimal.
RUN apt-get update \
 && apt-get install -y --no-install-recommends \
      ca-certificates \
      curl \
      gnupg \
      git \
      jq \
      gettext-base \
      unzip \
      tar \
      xz-utils \
 && rm -rf /var/lib/apt/lists/*

# kubectl
RUN curl -fsSL "https://dl.k8s.io/release/${KUBECTL_VERSION}/bin/linux/${TARGETARCH}/kubectl" -o /usr/local/bin/kubectl \
 && chmod 0755 /usr/local/bin/kubectl \
 && kubectl version --client=true

# helm
RUN curl -fsSL "https://get.helm.sh/helm-${HELM_VERSION}-linux-${TARGETARCH}.tar.gz" \
      | tar -xz -C /tmp \
 && mv "/tmp/linux-${TARGETARCH}/helm" /usr/local/bin/helm \
 && chmod 0755 /usr/local/bin/helm \
 && rm -rf "/tmp/linux-${TARGETARCH}" \
 && helm version --short

# yq for YAML manipulation in workflows
RUN curl -fsSL "https://github.com/mikefarah/yq/releases/download/${YQ_VERSION}/yq_linux_${TARGETARCH}" \
      -o /usr/local/bin/yq \
 && chmod 0755 /usr/local/bin/yq \
 && yq --version

# buildctl. The BuildKit client. We do not install buildkitd. Jobs target the in-cluster
# BuildKit Deployment via BUILDKIT_HOST.
RUN curl -fsSL "https://github.com/moby/buildkit/releases/download/${BUILDKIT_VERSION}/buildkit-${BUILDKIT_VERSION}.linux-${TARGETARCH}.tar.gz" \
      | tar -xz -C /tmp \
 && mv /tmp/bin/buildctl /usr/local/bin/buildctl \
 && chmod 0755 /usr/local/bin/buildctl \
 && rm -rf /tmp/bin \
 && buildctl --version

# Docker CLI only (no daemon). Useful for tagging and inspecting images locally.
RUN install -m 0755 -d /etc/apt/keyrings \
 && curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg \
 && chmod a+r /etc/apt/keyrings/docker.gpg \
 && echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo $VERSION_CODENAME) stable" \
      > /etc/apt/sources.list.d/docker.list \
 && apt-get update \
 && apt-get install -y --no-install-recommends docker-ce-cli \
 && rm -rf /var/lib/apt/lists/* \
 && docker --version

# Node.js 20 from NodeSource
RUN curl -fsSL "https://deb.nodesource.com/setup_${NODE_MAJOR}.x" | bash - \
 && apt-get install -y --no-install-recommends nodejs \
 && rm -rf /var/lib/apt/lists/* \
 && node --version \
 && npm --version

# Drop back to the unprivileged runner user. ARC expects this.
USER runner
WORKDIR /home/runner

# A small sanity message at container start. The ARC runner entrypoint takes over.
LABEL org.opencontainers.image.source="https://github.com/ajugulum123/KubernetesDeploymentFifaAPI"
LABEL org.opencontainers.image.description="Custom ARC runner image with kubectl, helm, buildctl, docker CLI, jq, yq and Node 20"
LABEL org.opencontainers.image.licenses="MIT"
