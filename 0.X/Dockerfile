FROM golang:1.10.3-alpine3.8 as build


ENV BUILD_ROOT="$GOPATH/src/github.com/hashicorp"
ENV BUILD_TAG="v1.2.2"

# tput requirement
ENV TERM=dumb

RUN set -x && \
    apk update && \
    apk add --no-cache \
    bash \
    git \
    make \
    ncurses && \
    mkdir -p $BUILD_ROOT && cd $BUILD_ROOT && \
    git clone -b $BUILD_TAG https://github.com/hashicorp/consul.git && cd consul && \
    make tools && make linux && \
    # tiny smoke test to ensure the binary we built runs
    /go/bin/consul -v

FROM alpine:3.7
LABEL maintainer="Mark Jeromin <mark.jeromin@sysfrog.net>"

# Create a consul user and group first so the IDs get set the same way, even as
# the rest of this may change over time.
RUN set -x && \
    addgroup consul && \
    adduser -S -G consul consul

# Set up certificates, base tools, and Consul.
RUN set -eux && \
    apk add --no-cache ca-certificates dumb-init libcap openssl su-exec

COPY --from=build /go/bin/consul /bin/consul

# The /consul/data dir is used by Consul to store state. The agent will be started
# with /consul/config as the configuration directory so you can add additional
# config files in that location.
RUN mkdir -p /consul/data && \
    mkdir -p /consul/config && \
    chown -R consul:consul /consul

# set up nsswitch.conf for Go's "netgo" implementation which is used by Consul,
# otherwise DNS supercedes the container's hosts file, which we don't want.
RUN test -e /etc/nsswitch.conf || echo 'hosts: files dns' > /etc/nsswitch.conf

# Expose the consul data directory as a volume since there's mutable state in there.
VOLUME /consul/data

# Server RPC is used for communication between Consul clients and servers for internal
# request forwarding.
EXPOSE 8300

# Serf LAN and WAN (WAN is used only by Consul servers) are used for gossip between
# Consul agents. LAN is within the datacenter and WAN is between just the Consul
# servers in all datacenters.
EXPOSE 8301 8301/udp 8302 8302/udp

# HTTP and DNS (both TCP and UDP) are the primary interfaces that applications
# use to interact with Consul.
EXPOSE 8500 8600 8600/udp

# Consul doesn't need root privileges so we run it as the consul user from the
# entry point script. The entry point script also uses dumb-init as the top-level
# process to reap any zombie processes created by Consul sub-processes.
COPY docker-entrypoint.sh /usr/local/bin/docker-entrypoint.sh
ENTRYPOINT ["/usr/local/bin/docker-entrypoint.sh"]

# By default you'll get an insecure single-node development server that stores
# everything in RAM, exposes a web UI and HTTP endpoints, and bootstraps itself.
# Don't use this configuration for production.
CMD ["agent", "-dev", "-client", "0.0.0.0"]
