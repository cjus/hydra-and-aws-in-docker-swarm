# Configuration management

## upload config.json

```shell
$ hydra-cli use gemini
$ hydra-cli cfg hydra-router:0.15.4 config.json
```

## Sample config

```javascript
{
  "cluster": false,
  "processes": 0,
  "externalRoutes": {},
  "risingStack": false,
  "debugLogging": true,
  "hydra": {
    "serviceName": "hydra-router",
    "serviceDescription": "Service Router",
    "serviceIP": "",
    "servicePort": 80,
    "serviceType": "router",
    "plugins": {
      "logger": {
        "logRequests": true,
        "redact": ["password"],
        "elasticsearch": {
          "host": "10.0.0.56",
          "port": 9200,
          "index": "hydra"
        }
      }
    },
    "redis": {
      "url": "redis://10.0.0.154:6379/15"
    }
  },
  "queuer": {
    "url": "10.0.0.154",
    "port": 6379,
    "db": 3
  }
}
```
