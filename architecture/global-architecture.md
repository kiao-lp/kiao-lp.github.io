# Global architecture

The goal of this document is to provide a minimalist but up-to-date overview of the architecture of KIAO.

## 1. Functionnal

```mermaid
flowchart TB
    %% Main zoning
    subgraph Note["Note"]
        N1["Create notes"]
        N2["Organize notes"]
        N3["Edit notes"]
    end

    subgraph Publication["Publication"]
        P1["Convert notes into units and publish them"]
    end

    subgraph Catalog["Catalog"]
        C1["Search for units"]
        C2["Subscribe to units"]
        C3["Unsubscribe from units"]
    end

    subgraph Account["Account"]
        U1["Register"]
        U2["Edit account"]
        U3["Authenticate"]
    end

    subgraph Learning["Learning"]
        E1["Learn"]
        E2["Customize training per unit"]
    end

    %% Links between zones
    Note -- Publish --> Publication
    Publication -- Publish --> Catalog
    Catalog -- Synchronization --> Learning
    Account -- Authentication --> Learning
    Account -- Authentication --> Catalog
```

## 2. Application

### Overview

```plantuml
@startuml
skinparam componentStyle rectangle
skinparam defaultTextAlignment center
left to right direction

package "Exchange" {
    node "Gateway (Nginx)" as GW
    component "<< frontend >>\nAccount" as MFCompte
    component "<< frontend >>\nNote" as MFNote
    component "<< frontend >>\nCatalog" as MFCatalog
    component "<< android >>\nLearning" as LearningAA

    MFCompte --> GW
    MFNote --> GW
    MFCatalog --> GW
    LearningAA --> GW
}

package "Operation" {

    rectangle "Account" as ZCompte
    rectangle "Note" as ZNote
    rectangle "Catalog" as ZCatalog

    component "Publication" as Publication

    queue "MOM (Kafka)" as MOM

    ZCompte --> MOM
    ZNote --> MOM
    ZCatalog --> MOM
    Publication --> MOM
}

package "Reference" {
    rectangle "Messages"
    rectangle "Shared Data"
    rectangle "MLScript"

    ZCompte --> Messages
    ZNote --> Messages
    ZCatalog --> Messages
}

package "Data Repository" {
    database "Account DB"
    database "Note DB"
    database "Unit DB"

    ZCompte --> "Account DB"
    ZNote --> "Note DB"
    ZCatalog --> "Unit DB"
}

GW --> ZCompte
GW --> ZNote
GW --> ZCatalog
@enduml
```

### Deployment

```plantuml
@startuml
skinparam componentStyle rectangle
skinparam defaultTextAlignment center

node "Web Client (browser)" {
  artifact "Frontend React/TS"
}

node "Android Client" {
  artifact "Android App"
}

cloud "Cloud" {

  ' --- Production Server ---
  node "VPS OVH\nkiao-lp.app (Ubuntu 24.10)" {
    
    node "Nginx (gateway)" {
      rectangle "reverse proxy"
    }

    node "Docker Compose" {
      component "account" <<service>>
      component "note" <<service>>
      component "unit" <<service>>
      component "publication" <<service>>

      component "PostgreSQL" {
        database "DB_account"
        database "DB_note"
        database "DB_catalog"
      }

      queue "Kafka"
    }
  }

  artifact "Google TTS API"
}

' --- Backup Zone ---
folder "External Backup" {
  database "PostgreSQL Backup"
}

' --- Network Communications ---

"Frontend React/TS" --> "reverse proxy" : HTTPS
"Android App" --> "reverse proxy" : HTTPS

"reverse proxy" --> "account"
"reverse proxy" --> "note"
"reverse proxy" --> "unit"

' --- Kafka ---
"note" -- "Kafka"
"account" -- "Kafka"
"unit" -- "Kafka"
"publication" -- "Kafka"

"publication" --> "Google TTS API"

' --- Databases ---
"account" --> "DB_account"
"note" --> "DB_note"
"unit" --> "DB_catalog"

PostgreSQL --> "PostgreSQL Backup" : replication/backup

@enduml
```

### Typical service organization

```plantuml
@startuml
skin rose
hide empty members

package microservice {

  package core {
  
    package business {
    
      package entities {
        class Entity {}
      }

      package services {
        class Service {}
      }
    }

    package ports {
      interface Repository {}

      interface Producer {}
    }
  }

  package adapters {

    package data {
      class JpaRepository {}

    }

    package kafka {
      class KafkaProducer {}
      class KafkaConsumer {}
    }

    package rest {
      class Controller {}
    }
  }

  JpaRepository ..|> core.ports.Repository
  KafkaProducer ..|> core.ports.Producer
  Controller --> Service
  KafkaConsumer --> Service
  Service --> Repository
  Service --> Producer
}
@enduml
```

#### Rules

- **adapters are interchangeable** and activated by configuration. For example, a kafka producter can be replaced by a HTTP request sended without impacting the core.
