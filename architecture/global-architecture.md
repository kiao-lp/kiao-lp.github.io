# Global architecture

The goal of this document is to provide a minimalist but up-to-date overview of the architecture of KIAO-Web, the web extension of KIAO.

## 1. Zoning

```plantuml
@startuml
skinparam componentStyle rectangle
skinparam defaultTextAlignment center
left to right direction

' ---- EXCHANGE ZONE (EXPOSURE) ----
package "Exchange Zone" {
    node "Gateway (Nginx)" as GW
    component "Micro-frontend\naccount" as MFCompte
    component "Micro-frontend\nnote" as MFNote
    component "Micro-frontend\nunit" as MFUnite

    MFCompte --> GW
    MFNote --> GW
    MFUnite --> GW
}

' ---- GROUPED OPERATIONAL ZONES ----
package "Operational Zones" {

    rectangle "Account" as ZCompte
    rectangle "Note" as ZNote
    rectangle "Unit" as ZUnite

    ' ---- HYBRID SERVICE ----
    component "Unit Builder" as UnitBuilder

    ' ---- MOM between operational zones ----
    queue "MOM (Kafka)" as MOM

    ZCompte --> MOM
    ZNote --> MOM
    ZUnite --> MOM
    UnitBuilder --> MOM
}

' ---- REFERENCE ZONE ----
package "Reference Zone" {
    rectangle "Messages"
    rectangle "Shared Data"
    rectangle "MLScript"

    ZCompte --> Messages
    ZNote --> Messages
    ZUnite --> Messages
}

' ---- DATA REPOSITORY ZONE ----
package "Data Repository Zone" {
    database "Account DB"
    database "Note DB"
    database "Unit DB"

    ZCompte --> "Account DB"
    ZNote --> "Note DB"
    ZUnite --> "Unit DB"
}

' ---- API RELATIONS ----
GW --> ZCompte
GW --> ZNote
GW --> ZUnite
@enduml
```

### 2. Deployment

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
      component "unit-builder" <<service>>

      component "PostgreSQL" {
        database "DB_account"
        database "DB_note"
        database "DB_unit"
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
"unit-builder" -- "Kafka"

"unit-builder" --> "Google TTS API"

' --- Databases ---
"account" --> "DB_account"
"note" --> "DB_note"
"unit" --> "DB_unit"

PostgreSQL --> "PostgreSQL Backup" : replication/backup

@enduml
```

### 3. Typical service organization

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
