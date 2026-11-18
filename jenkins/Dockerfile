FROM jenkins/jenkins:2.591@sha256:46336574d4d0fbced8679901cf47b1b15f6ca385188e0ec5717fb3b11c92ebee

LABEL org.opencontainers.image.title="Jenkins - Docker Security Pipeline"

USER root

# Trivy + client Docker (uniquement le CLI, pas le démon)
RUN apt-get update \
 && apt-get install -y --no-install-recommends wget gnupg ca-certificates docker-cli \
 && wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | gpg --dearmor -o /usr/share/keyrings/trivy.gpg \
 && echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb generic main" > /etc/apt/sources.list.d/trivy.list \
 && apt-get update \
 && apt-get install -y --no-install-recommends trivy \
 && rm -rf /var/lib/apt/lists/*

USER jenkins

# Plugins nécessaires au pipeline
RUN jenkins-plugin-cli --plugins \
    workflow-aggregator \
    git \
    sonar \
    credentials-binding \
    htmlpublisher