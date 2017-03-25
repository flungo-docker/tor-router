FROM alpine:3.5

# Install the required packages
RUN apk add --update tor iptables sudo

# Expose the ports made available through Tor
EXPOSE 9001 9040 9050 5353/udp

# Populate the tor-router configurations
COPY . /opt/tor-router

# Set the container's WORKDIR and ENTRYPOINT
WORKDIR /opt/tor-router
ENTRYPOINT ["/opt/tor-router/entrypoint.sh"]
