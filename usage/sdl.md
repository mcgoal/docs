# Stack Definition Language

## Deployment Configuration

Deployment services, datacenters, pricing, etc.. are described by a [YAML](http://www.yaml.org/start.html) configuration file. Configuration files may end in `.yml` or `.yaml`.

A complete deployment has the following sections:

* [version](sdl.md#version)
* [services](sdl.md#services)
* [profiles](sdl.md#profiles)
* [deployment](sdl.md#deployment)

### Example

```yaml
---
version: "1.0"

include:
  - "foo.yml"
  - "https://foo.yml"

services:

  db-master:
    image: postgres
    expose:
      - port: 5432
        proto: tcp
        to:
          - service: db-pool
          - service: db-pool
            global:  true
          - service: db-slave
            global: true

  db-slave:
    image: postgres-slave
    depends-on:
      - service: db-master
    expose:
      - port: 5432
        proto: tcp
        to:
          - service: db-pool

  db-pool:
    image: db-pool
    depends-on:
      - service: db-slave
      - service: db-master
    expose:
      - port: 5432
        proto: tcp
        to:
          - service: web

  web:
    image: foo:latest
    env:
      - "API_KEY=996fb92427ae41e4649b934ca495991b785"
    port: 80
    depends-on:
      - service: db-pool
    expose:
      - port: 443
        as: 8080
        accept:
          - foo.com
        to:
          - global: true

profiles:

  compute:
    web:
      cpu: "0.01"
      memory: "128Mi"
      storage: "512Mi"
    db:
      cpu: "0.01"
      memory: "128Mi"
      storage: "1Gi"
    db-pool:
      cpu: "0.01"
      memory: "128Mi"
      storage: "512Mi"

  placement:
    westcoast:
      attributes:
        region: us-west
      pricing:
        web: 10u
        db:  50u
        db-pool: 30u
    eastcoast:
      attributes:
        region: us-east
      pricing:
        web: 30u
        db:  60u
        db-pool: 40u

deployment:

  db-master:
    westcoast:
      profile: db
      count: 1

  db-slave:
    westcoast:
      profile: db
      count: 1
    eastcoast:
      profile: db
      count: 1

  db-pool:
    westcoast:
      profile: db-pool
      count: 1
    eastcoast:
      profile: db-pool
      count: 1

  web:
    westcoast:
      profile: web
      count: 2
    eastcoast:
      profile: web
      count: 2
```

An example deployment configuration for the Akash testnet can be found [here](https://github.com/ovrclk/docs/tree/26be854c4086a1987a9960324ae30bb336b0e8c2/sdl/testnet/testnet-deployment.yml). A full example deployment configuration can be found [here](https://github.com/ovrclk/docs/tree/26be854c4086a1987a9960324ae30bb336b0e8c2/sdl/deployment.yml).

## version

Indicates version of Akash configuration file. Currently only `"1.0"` is accepted.

## services

The top-level `services` entry contains a map of workloads to be ran on the Akash deployment. Each key is a service name; values are a map containing the following keys:

| Name | Required | Meaning |
| :--- | :--- | :--- |
| `image` | Yes | Docker image of the container |
| `depends-on` | No | List of services which must be brought up before the current service |
| `args` | No | Arguments to use when executing the container |
| `env` | No | Environment variables to set in running container |
| `expose` | No | Entities allowed to connect to to the services.  See [services.expose](sdl.md#servicesexpose). |

## services.expose

`expose` is a list describing what can connect to the service. Each entry is a map containing one or more of the following fields:

| Name | Required | Meaning |
| :--- | :--- | :--- |
| `port` | Yes | Container port to expose |
| `as` | No | Port number to expose the container port as |
| `accept` | No | List of hosts to accept connections for |
| `proto` | No | Protocol type \(`tcp`,`http`, or `https`\) |
| `to` | No | List of entities allowed to connect.  See [services.expose.to](sdl.md#servicesexposeto) |

The `port` value governs the default `proto` value as follows:

| `port` | `proto` default |
| :--- | :--- |
| 80 | http |
| 443 | https |
| all others | tcp |

### services.expose.to

`expose.to` is a list of clients to accept connections from. Each item is a map with one or more of the following entries:

| Name | Value | Default | Description |
| :--- | :--- | :--- | :--- |
| `service` | A service in this deployment |  | Allow the given service to connect |
| `global` | `true` or `false` | `false` | If true, allow connections from outside of the datacenter |

If no service is given and `global` is true, any client can connect from anywhere \(web servers typically want this\).

If a service name is given and `global` is `false`, only the services in the current datacenter can connect. If a service name is given and `global` is `true`, services in other datacenters for this deployment can connect.

If `global` is `false` then a service name must be given.

## profiles

The `profiles` section contains named compute and placement profiles to be used in the [deployment](sdl.md#deployment).

### profiles.compute

`profiles.compute` is map of named compute profiles. Each profile specifies compute resources to be leased for each service instance uses uses the profile.

Example:

This defines a profile named `web` having resource requirements of 2 vCPUs, 2 gigabytes of memory, and 5 gigabytes of storage available.

```yaml
web:
  cpu: 2
  memory: "2Gi"
  storage: "5Gi"
```

`cpu` units represent a vCPU share and can be fractional. When no suffix is present the value represents a fraction of a whole CPU share. With a `m` suffix, the value represnts the number of milli-CPU shares \(1/1000 of a CPU share\).

Example:

| Value | CPU-Share |
| :--- | :--- |
| `1` | 1 |
| `0.5` | 1/2 |
| `"100m"` | 1/10 |
| `"50m"` | 1/20 |

`memory`, `storage` units are described in bytes. The following suffixes are allowed for simplification:

| Suffix | Value |
| :--- | :--- |
| `k` | 1000 |
| `Ki` | 1024 |
| `M` | 1000^2 |
| `Mi` | 1024^2 |
| `G` | 1000^3 |
| `Gi` | 1024^3 |
| `T` | 1000^4 |
| `Ti` | 1024^4 |
| `P` | 1000^5 |
| `Pi` | 1024^5 |
| `E` | 1000^6 |
| `Ei` | 1024^6 |

### profiles.placement

`profiles.placement` is map of named datacenter profiles. Each profile specifies required datacenter attributes and pricing configuration for each [compute profile](sdl.md#profilescompute) that will be used within the datacenter.

Example:

```yaml
westcoast:
  attributes:
    region: us-west
  pricing:
    web: 8u
    db: 100u
```

This defines a profile named `westcoast` having required attributes `{region="us-west"}`, and with a max price for the `web` and `db` [compute profiles](sdl.md#profilescompute) of 8 and 15 _micro_ \(10^-6\) tokens per block, respectively.

Pricing may be expressed in decimal or scientific notation for Akash units, or may be suffixed with `mu`,`µ`, or `u` to represent _micro_ Akash.

Examples:

| Value | Micro Akash Tokens |
| :--- | :--- |
| `1` | 1000000 |
| `1e-4` | 100 |
| `20u` | 20 |

## deployment

The `deployment` section defines how to deploy the services. It is a mapping of service name to deployment configuration.

Each service to be deployed has an entry in the `deployment`. This entry is maps [datacenter profiles](sdl.md#profilesplacement) to [compute profiles](sdl.md#profilescompute) to create a final desired configuration for the resources required for the service.

Example:

```yaml
web:
  westcoast:
    profile: web
    count: 20
```

This says that the 20 instances of the `web` service should be deployed to a datacenter matching the `westcoast` [datacenter profile](sdl.md#profilesplacement). Each instance will have the resources defined in the `web` [compute profile](sdl.md#profilescompute) available to it.

