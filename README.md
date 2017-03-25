# Docker Tor Proxy/Router

A tor proxy/router allowing Docker containers to use Tor networking. The image provides both a SOCKS proxy as well as transparent routing configuration making it extremely versatile for a variety of applications and use-cases. The container only supports TCP traffic and will block all non-TCP traffic (except UDP DNS which will be intercepted).

**Warning:** Do not run this container with `--net=host`. Doing so may change your host networking stack such that you will not be able to access the internet. In a future version, we will try to detect this and kill the container.

## Usage

In all methods, it is required that the container is started with `--cap-add NET_ADMIN`, this is to allow the configuration of firewall rules that restrict outbound traffic and configure transparent routing of TCP traffic.

To start the container run the following command:

```
docker run -d --name tor-router --cap-add NET_ADMIN --dns 127.0.0.1 flungo/tor
```

The `NET_ADMIN` capability is used to allow configuration of the iptables.

To use this router, several methods can be employed for different use-cases:

- [Proxy](#proxy)
- [Routing with shared network stack](#routing-with-shared-network-stack)
- [Routing over Docker network](#routing-over-docker-network)

### Proxy

The simplest use-case is to use the container as SOCKS proxy. As well as the SOCKS proxy, you should also use the DNS server provided by the container exclusively to ensure that your identity is not revealed by DNS leakage as well as being able to resolve `.onion` address for applications that don't support using the SOCKS proxy for DNS queries.

#### Proxy on host machine

To access the proxy from the host machine or via another machine on the network, the router can either be run with the SOCKS proxy and or accessible directly via its IP if routing is configured. To publish the SOCKS proxy and DNS server on ports `9050` and `53` respectively, the following command can be used:

```
docker run -d \
  --name tor-router \
  --cap-add NET_ADMIN \
  --dns 127.0.0.1 \
  -p 9050:9050 \
  -p 53:5353/udp \
  flungo/tor
```

From the host machine you can then use the SOCKS proxy in conjunction with the DNS server to route traffic over the Tor network. The DNS server will need to be set in `/etc/resolve.conf` (unless the application you wish to protect allows the DNS to be configured). Here is an example of using curl on the host machine through the proxy:

```
curl --proxy socks5h://<router_ip>:9050 https://check.torproject.org/api/ip
```

When using in this scenario there are several potential ways that packets could leak from being routed on the Tor network so care should be taken to avoid these. For anominity, one of the other methods is recommended.

#### Proxy from container

The proxy can be accessed by another container by creating a network which both the container and the router are attached to. For the avoidance of packet leakage, it is recommended that an isolated network is created:

```
docker network create --internal tor
```

It is then required that you connect the routing container to the network:

```
docker network connect tor tor-router
```

The IP address assigned to the router within this network can then be obtained using:

```
docker inspect -f '{{ .NetworkSettings.Networks.tor.IPAddress}}' tor-router
```

It may be worth considering giving the network and router a fixed IP range and address respectively, but this is not covered in this example.

When running the container, the SOCKS configuration will vary depending on the application which is being run. This example shows the socks proxy being used with cURL, but for your application, it may require environment variables, command line flags or modifications to configuration files. In this example `<router_ip>` should be replaced with the IP address of the router within the network.

```
docker run --rm --net=tor --dns <router_ip> flungo/netutils \
  curl --proxy socks5h://<router_ip>:9050 https://check.torproject.org/api/ip
```

### Routing with shared network stack

It is not recommended, but it is possible to start a new container and share the tor-router's networking stack. This can be done using `--net=container:tor-router`. For example:

```
docker run --rm --net=container:tor-router flungo/netutils \
  curl https://check.torproject.org/api/ip
```

The new container will share the networking stack of the tor-router which has been configured to transparently route all outbound TCP traffic via the Tor Proxy.

**Warning:** Containers attached to the router in this way should never be given `--privileged` or `--cap-add NET_ADMIN`. If the container runs any process with UID `9001`, this process will completely bypass the transparent routing.

### Routing over Docker network

A container routing driver is currently in development. When this is complete, it will be possible to create Docker networks that utilise this image as a router, at which time this documentation will be updated with the relevant instructions.

## Notes

- `10.192.0.0/10` is used as the range for `.onion` addresses and should not be used on any attached network.
- Setting `--dns 127.0.0.1` for the router is not required but makes configuration of containers attached by [routing with shared network stack](#routing-with-shared-network-stack) simpler.
- All UDP DNS requests will be handled by the Tor DNS resolver and will be able to resolve `.onion` addresses to a virtual address in the range `10.192.0.0/10`.
- TCP DNS requests will be allowed and will be routed through the Tor network but will not be able to resolve `.onion` addresses.
