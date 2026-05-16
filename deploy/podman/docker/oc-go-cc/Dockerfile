FROM alpine:3.21 AS download
ARG TARGETARCH
RUN apk add --no-cache curl
ARG OC_GO_CC_VERSION=v0.1.3
RUN case "$TARGETARCH" in \
      amd64) BIN=oc-go-cc_linux-amd64 ;; \
      arm64) BIN=oc-go-cc_linux-arm64 ;; \
      *) echo "unsupported arch: $TARGETARCH"; exit 1 ;; \
    esac && \
    curl -fsSL -o /bin/oc-go-cc \
      "https://github.com/samueltuyizere/oc-go-cc/releases/download/${OC_GO_CC_VERSION}/${BIN}" && \
    chmod +x /bin/oc-go-cc

FROM scratch
ENV HOME=/root
COPY --from=download /bin/oc-go-cc /bin/oc-go-cc
COPY --from=download /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
COPY config.json /root/.config/oc-go-cc/config.json
ENTRYPOINT ["/bin/oc-go-cc", "serve"]
