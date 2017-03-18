# Creating services

```shell
$ docker service create \
    --name hydra-router \
    --network servicenet \
    --constraint=node.role==manager \
    --env HYDRA_REDIS_URL="redis://10.0.0.154:6379/15" \
    --env HYDRA_SERVICE="hydra-router:0.15.4" \
    --publish 80:80 \
    --replicas=1 \
    flywheelsports/fwsp-hydra-router:0.15.6
```

> When a service is created which using `--publish` it is automatically added to the `ingress` network.

```shell
$ docker service create \
    --name hello-service \
    --network servicenet \
    --constraint=node.role==worker \
    --env HYDRA_REDIS_URL="redis://10.0.0.154:6379/15" \
    --env HYDRA_SERVICE="hello-service:0.0.2" \
    --replicas=1 \
    cjus/hello-service:0.0.5
```

> Creating a service which does not use `--publish` places the service in the `servicenet`.  The service can still listen on a port for inter-service communication.

## Removing services

```shell
$ docker service rm hydra-router
$docker service rm hello-service
```
