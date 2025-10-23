## Overview

This repository contains a GitOps-managed Kubernetes homelab cluster running k3s on Ubuntu nodes. The cluster hosts home automation, media services, databases, monitoring, and various web applications.

### Key Technologies

- **Kubernetes**: k3s lightweight distribution
- **GitOps**: Flux CD for continuous delivery
- **Networking**: Cilium CNI with Gateway API
- **Storage**: Longhorn distributed block storage
- **Database**: CloudNative-PG (PostgreSQL), MariaDB, Redis (Dragonfly)
- **Secrets**: SOPS encryption with Age/GPG
- **Certificates**: cert-manager for TLS

## Architecture

### Infrastructure Layer

```mermaid
graph TB
    Internet[Internet] -->|Gateway API| Cilium[Cilium CNI + Gateway API]
    HomeNetwork[Home Network] -->|Local Access| Cilium

    Cilium -->|Routes Traffic| Apps[Applications]

    CertManager[cert-manager] -->|TLS Certificates| Cilium
    Longhorn[Longhorn Storage] -->|Block Storage| Apps

    NFS[NFS Storage] -.->|Media & Photos| Apps

    style Cilium fill:#9cf,stroke:#333,stroke-width:3px
    style Longhorn fill:#fc9,stroke:#333,stroke-width:2px
    style CertManager fill:#6c3,stroke:#333,stroke-width:2px
```

### Data Services

```mermaid
graph LR
    subgraph "Database Services"
        PSQL[CloudNative-PG<br/>PostgreSQL]
        MariaDB[MariaDB]
        Redis[Dragonfly<br/>Redis]
    end

    subgraph "Consumers"
        HA[Home Assistant]
        Immich[Immich]
        Ghost[Ghost]
        Ghostfolio[Ghostfolio]
        Miniflux[Miniflux]
    end

    HA --> PSQL
    Immich --> PSQL
    Ghostfolio --> PSQL
    Miniflux --> PSQL
    Ghost --> MariaDB

    style PSQL fill:#336791,color:#fff,stroke:#333,stroke-width:2px
    style MariaDB fill:#003545,color:#fff,stroke:#333,stroke-width:2px
    style Redis fill:#dc382d,color:#fff,stroke:#333,stroke-width:2px
```

### Application Services

```mermaid
graph TB
    subgraph "Home Automation"
        HA[Home Assistant]
        ESPHome[ESPHome]
        Z2M[Zigbee2MQTT]
        Mosquitto[MQTT Broker]
        Frigate[Frigate NVR]
    end

    subgraph "Media & Photos"
        Plex[Plex]
        Immich[Immich]
        Sonarr[Sonarr]
        Radarr[Radarr]
        Overseerr[Overseerr]
    end

    subgraph "Productivity"
        Ghost[Ghost Blog]
        Miniflux[Miniflux RSS]
        Ghostfolio[Ghostfolio]
        Glance[Glance Dashboard]
    end

    subgraph "Monitoring"
        Prometheus[Prometheus]
        Grafana[Grafana]
        Loki[Loki]
    end

    style HA fill:#41bdf5,stroke:#333,stroke-width:2px
    style Plex fill:#e5a00d,stroke:#333,stroke-width:2px
    style Immich fill:#4250af,color:#fff,stroke:#333,stroke-width:2px
    style Grafana fill:#f46800,stroke:#333,stroke-width:2px
```

### Storage Architecture

```mermaid
graph TB
    subgraph "Storage Backends"
        NodeDisks[Node Local Disks]
        NFS[NFS Server]
    end

    subgraph "Longhorn Distributed Storage"
        LH[Longhorn Controller]
        LH -->|Replicates across nodes| Volumes[Replicated Volumes]
    end

    subgraph "Longhorn Consumers"
        PSQL[(PostgreSQL<br/>Databases)]
        Config[(Application<br/>Configs)]
        HA_Data[(Home Assistant<br/>Data)]
    end

    subgraph "NFS Consumers"
        Media[(Plex Media<br/>Library)]
        Photos[(Immich<br/>Photos)]
    end

    NodeDisks --> LH
    Volumes -->|PVC| PSQL
    Volumes -->|PVC| Config
    Volumes -->|PVC| HA_Data

    NFS -->|NFS Mount| Media
    NFS -->|NFS Mount| Photos

    style LH fill:#fc9,stroke:#333,stroke-width:2px
    style PSQL fill:#336791,color:#fff,stroke:#333,stroke-width:2px
    style NFS fill:#6c3,stroke:#333,stroke-width:2px
```

### Home Automation Stack

```mermaid
graph LR
    subgraph "Devices"
        Zigbee[Zigbee Devices]
        ESP[ESP32/ESP8266]
        Cameras[IP Cameras]
    end

    subgraph "Integration Layer"
        Z2M[Zigbee2MQTT]
        ESPHome[ESPHome]
        Frigate[Frigate NVR]
    end

    subgraph "Core"
        MQTT[Mosquitto MQTT]
        HA[Home Assistant]
    end

    subgraph "Storage"
        PSQL[(PostgreSQL)]
        ConfigPVC[(Config PVCs)]
    end

    Zigbee --> Z2M
    ESP --> ESPHome
    Cameras --> Frigate

    Z2M --> MQTT
    Frigate --> MQTT
    ESPHome --> HA
    MQTT --> HA

    HA --> PSQL
    HA --> ConfigPVC
    Z2M --> ConfigPVC
    Frigate --> ConfigPVC

    style HA fill:#41bdf5,stroke:#333,stroke-width:3px
    style MQTT fill:#660066,color:#fff,stroke:#333,stroke-width:2px
```

### Media Automation Stack

```mermaid
graph TD
    User[User Requests] --> Overseerr[Overseerr]

    Overseerr -->|TV Request| Sonarr
    Overseerr -->|Movie Request| Radarr

    Prowlarr[Prowlarr Indexer] -.->|Indexers| Sonarr
    Prowlarr -.->|Indexers| Radarr

    Sonarr -->|Download| Media[(Media Storage)]
    Radarr -->|Download| Media

    Bazarr[Bazarr Subtitles] -->|Subtitles| Sonarr
    Bazarr -->|Subtitles| Radarr

    Media --> Plex[Plex Media Server]
    Plex --> Viewers[Viewers]

    style Overseerr fill:#a368fc,stroke:#333,stroke-width:2px
    style Plex fill:#e5a00d,stroke:#333,stroke-width:3px
```

## :open_file_folder:&nbsp; Repository structure

The Git repository contains the following directories under `cluster` and are ordered below by how Flux will apply them.

- **base** directory is the entrypoint to Flux
- **crds** directory contains custom resource definitions (CRDs) that need to exist globally in your cluster before anything else exists
- **core** directory (depends on **crds**) are important infrastructure applications (grouped by namespace) that should never be pruned by Flux
- **apps** directory (depends on **core**) is where your common applications (grouped by namespace) could be placed, Flux will prune resources here if they are not tracked by Git anymore

```
cluster
├── apps
│   ├── default
│   ├── networking
│   └── system-upgrade
├── base
│   └── flux-system
├── core
│   ├── cert-manager
│   ├── metallb-system
│   ├── namespaces
│   └── system-upgrade
└── crds
    └── cert-manager
```
