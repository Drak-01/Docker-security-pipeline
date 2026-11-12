FROM jenkins/jenkins:lts-jdk21@sha256:4c03015d601fe7a761220979d2dd7bf465091e2d13de775eb1980627811b3aaa

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