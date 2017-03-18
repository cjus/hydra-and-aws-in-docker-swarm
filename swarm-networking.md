## Swarm networking

```shell
$ docker network create servicenet \
  --driver overlay \
  --subnet 10.0.9.0/24
```

```shell
$ docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
7ffba041b5b9        bridge              bridge              local
90d25bc2e419        docker_gwbridge     bridge              local
7af9c7ddd8f6        host                host                local
p5f0bg197oia        ingress             overlay             swarm
e5f86522a1d0        none                null                local
z6vut7t9439u        servicenet          overlay             swarm
```

| network | usage | scope |
| --- | --- | --- |
| docker_gwbridge | A network created by Docker to allow containers to connect to their host | local |
| ingress | Network created by Docker to externally expose services and utilize the routing mesh | swarm |
| servicenet | Overlay network we created to enable container to container communication | swarm |


