FROM golang:1.12.5-stretch

ENV SRC_DIR /go/src/github.com/ipfs/go-ipfs

ENV TAG v0.5.1

RUN apt-get update && apt-get install -y wget ca-certificates
#RUN git clone --branch $BRANCH https://github.com/ipfs/go-ipfs.git $SRC_DIR

#COPY go.mod $SRC_DIR
#COPY go.sum $SRC_DIR

# Build the thing.
# Also: fix getting HEAD commit hash via git rev-parse.
# Also: allow using a custom IPFS API endpoint.
#RUN cd $SRC_DIR \
#  && mkdir -p .git/objects \
#  && go get -v ./cmd/... \
#  && GOPROXY='https://proxy.golang.org' make build


RUN mkdir -p /go/src/github.com/ipfs/go-ipfs && wget https://github.com/ipfs/go-ipfs/releases/download/${TAG}/go-ipfs_${TAG}_linux-amd64.tar.gz -O $SRC_DIR/go-ipfs_${TAG}_linux-amd64.tar.gz 
RUN cd $SRC_DIR && \
    tar -xzvf go-ipfs_${TAG}_linux-amd64.tar.gz


# Get su-exec, a very minimal tool for dropping privileges,
# and tini, a very minimal init daemon for containers
ENV SUEXEC_VERSION v0.2
ENV TINI_VERSION v0.16.1
RUN set -x \
  && cd /tmp \
  && git clone https://github.com/ncopa/su-exec.git \
  && cd su-exec \
  && git checkout -q $SUEXEC_VERSION \
  && make \
  && cd /tmp \
  && wget -q -O tini https://github.com/krallin/tini/releases/download/$TINI_VERSION/tini \
  && chmod +x tini

# Now comes the actual target image, which aims to be as small as possible.
FROM busybox:1-glibc

# Get the ipfs binary, entrypoint script, and TLS CAs from the build container.
ENV SRC_DIR /go/src/github.com/ipfs/go-ipfs
COPY --from=0 $SRC_DIR/go-ipfs/ipfs /usr/local/bin/ipfs
COPY --from=0 /tmp/su-exec/su-exec /sbin/su-exec
COPY --from=0 /tmp/tini /sbin/tini
COPY --from=0 /etc/ssl/certs /etc/ssl/certs

# This shared lib (part of glibc) doesn't seem to be included with busybox.
COPY --from=0 /lib/x86_64-linux-gnu/libdl-2.24.so /lib/libdl.so.2

# Ports for Swarm TCP, Swarm uTP, API, Gateway, Swarm Websockets
EXPOSE 4001
EXPOSE 4002/udp
EXPOSE 5001
EXPOSE 8080
EXPOSE 8081

# Create the fs-repo directory and switch to a non-privileged user.
ENV IPFS_PATH /data/ipfs
RUN mkdir -p $IPFS_PATH \
  && adduser -D -h $IPFS_PATH -u 1000 -G users ipfs \
  && chown ipfs:users $IPFS_PATH

# Expose the fs-repo as a volume.
# start_ipfs initializes an fs-repo if none is mounted.
# Important this happens after the USER directive so permission are correct.
VOLUME $IPFS_PATH

# The default logging level
ENV IPFS_LOGGING ""

COPY start_ipfs /usr/local/bin/start_ipfs

# This just makes sure that:
# 1. There's an fs-repo, and initializes one if there isn't.
# 2. The API and Gateway are accessible from outside the container.
ENTRYPOINT ["/sbin/tini", "--", "/usr/local/bin/start_ipfs"]

# Execute the daemon subcommand by default
CMD ["daemon", "--migrate=true"]