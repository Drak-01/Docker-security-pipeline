FROM ubuntu/squid:7.2-26.04_edge@sha256:42decc7f79b4e1044f4fe9b3a855af2cbfc76652f3191694726e21058443edc9

COPY squid.conf /etc/squid/squid.conf

HEALTHCHECK --interval=15s --timeout=3s --retries=5 \
  CMD timeout 2 bash -c '</dev/tcp/127.0.0.1/3128' || exit 1