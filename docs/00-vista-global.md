# 00 — Vista global del stack Wattify

Mapa de los 5 dominios funcionales del stack n8n Wattify y sus interacciones con servicios externos.

```mermaid
flowchart LR
    %% =========================================================
    %% Servicios externos / canales de entrada
    %% =========================================================
    subgraph EXT_IN["Canales de entrada"]
        direction TB
        Visitor((Visitante<br/>chat web)):::external
        Editor((Editor RRSS<br/>en Airtable)):::external
    end

    %% =========================================================
    %% Dominio: Asistente Isabel
    %% =========================================================
    subgraph ISABEL["Asistente Isabel  (tag: Agente conversacional)"]
        direction TB
        Chatwoot[["Chatwoot self-hosted<br/>inbox del agente"]]:::saas
        WF_CHAT["Chatwoot Widget Orquestador"]:::wf
        WF_RAG["RAG (cron 4h)"]:::wf
        WF_DISP["Cal.com Disponibilidad"]:::wf
        WF_AG["Cal.com Agendar"]:::wf

        WF_CHAT -.->|tool: consultar_disponibilidad| WF_DISP
        WF_CHAT -.->|tool: agendar_reunion| WF_AG
        WF_CHAT -.->|tool: base_conocimiento_wattify| WF_RAG
    end

    %% =========================================================
    %% Dominio: RRSS LinkedIn (post + imagen)
    %% =========================================================
    subgraph RRSS_LK["Pipeline RRSS LinkedIn  (tag: RRSS)"]
        direction TB
        Airtable[("Airtable<br/>base RRSS")]:::store
        WF_TXT_LK["Generar Texto Post LK"]:::wf
        WF_IMG_LK["Generar Imagen Post LK"]:::wf
        WF_PUB_LK["Publicar Post LK aprobado"]:::wf

        Editor -->|webhook generar texto| WF_TXT_LK
        Editor -->|webhook generar imagen| WF_IMG_LK
        Editor -->|webhook publicar (record_id)| WF_PUB_LK
        WF_TXT_LK --> Airtable
        WF_IMG_LK --> Airtable
        Airtable --> WF_PUB_LK
    end

    %% =========================================================
    %% Dominio: RRSS Video
    %% =========================================================
    subgraph RRSS_VID["Pipeline RRSS Video  (tag: RRSS)"]
        direction TB
        WF_TXT_VID["Crear texto video"]:::wf
        WF_AVATAR["Avatar Video v3"]:::wf

        Editor -->|webhook generar guión| WF_TXT_VID
        Editor -->|webhook generar avatar| WF_AVATAR
        WF_TXT_VID --> Airtable
        WF_AVATAR --> Airtable
    end

    %% =========================================================
    %% Dominio: RRSS Instagram Reels
    %% =========================================================
    subgraph RRSS_IG["Pipeline RRSS Instagram  (tag: Instagram)"]
        direction TB
        WF_TXT_IG["Generar Texto Reel IG"]:::wf
        WF_REEL_IG["Generar y Publicar Reel IG"]:::wf

        Editor -->|webhook generar texto reel| WF_TXT_IG
        Editor -->|webhook generar reel| WF_REEL_IG
        WF_TXT_IG --> Airtable
        Airtable --> WF_REEL_IG
        WF_REEL_IG --> Airtable
    end

    %% =========================================================
    %% Dominio: Infraestructura
    %% =========================================================
    subgraph INFRA["Infraestructura"]
        direction TB
        WF_ERR["Error Notifier Global<br/>(Error Trigger)"]:::wf
    end

    %% =========================================================
    %% Almacenes compartidos
    %% =========================================================
    subgraph STORES["Almacenes compartidos"]
        direction LR
        Redis[("Redis<br/>buffer + memory + idem + ratelimit")]:::store
        Postgres[("Postgres (pgvector)<br/>documents<br/>agent_turn_logs<br/>ingested_files")]:::store
    end

    %% =========================================================
    %% Servicios externos
    %% =========================================================
    subgraph SAAS["Servicios externos (APIs)"]
        direction LR
        GDrive[["Google Drive<br/>knowledge base"]]:::saas
        Gemini[["Google Gemini<br/>chat + imagen + embeddings"]]:::saas
        Cal[["Cal.com v2"]]:::saas
        LinkedIn[["LinkedIn REST API<br/>(organización)"]]:::saas
        HeyGen[["HeyGen v3<br/>avatares + voces"]]:::saas
        IG[["Instagram Graph API<br/>(cuenta Business)"]]:::saas
        Gmail[["Gmail<br/>(email operativo)"]]:::saas
    end

    %% =========================================================
    %% Entradas humanas a Isabel
    %% =========================================================
    Visitor -->|mensaje en widget| Chatwoot
    Chatwoot -->|webhook entrada| WF_CHAT

    %% =========================================================
    %% Conexiones Isabel ↔ resto
    %% =========================================================
    WF_CHAT --> Redis
    WF_CHAT --> Postgres
    WF_CHAT -->|extraer datos + actualizar contacto| Chatwoot
    WF_CHAT -->|email primer aviso + cita reservada| Gmail
    WF_CHAT --> Gemini
    WF_RAG --> GDrive
    WF_RAG --> Postgres
    WF_RAG --> Gemini
    WF_DISP --> Cal
    WF_AG --> Cal
    WF_AG --> Redis

    %% =========================================================
    %% Conexiones RRSS LK ↔ resto
    %% =========================================================
    WF_TXT_LK --> Gemini
    WF_IMG_LK --> Gemini
    WF_PUB_LK --> LinkedIn

    %% =========================================================
    %% Conexiones RRSS Video ↔ resto
    %% =========================================================
    WF_TXT_VID --> Gemini
    WF_AVATAR --> HeyGen
    WF_AVATAR --> LinkedIn

    %% =========================================================
    %% Conexiones RRSS Instagram ↔ resto
    %% =========================================================
    WF_TXT_IG --> Gemini
    WF_REEL_IG --> Gemini
    WF_REEL_IG --> HeyGen
    WF_REEL_IG --> IG

    %% =========================================================
    %% Error Notifier (configurado como errorWorkflow en los otros 11)
    %% =========================================================
    WF_CHAT -.->|on error| WF_ERR
    WF_RAG -.->|on error| WF_ERR
    WF_DISP -.->|on error| WF_ERR
    WF_AG -.->|on error| WF_ERR
    WF_TXT_LK -.->|on error| WF_ERR
    WF_IMG_LK -.->|on error| WF_ERR
    WF_PUB_LK -.->|on error| WF_ERR
    WF_TXT_VID -.->|on error| WF_ERR
    WF_AVATAR -.->|on error| WF_ERR
    WF_TXT_IG -.->|on error| WF_ERR
    WF_REEL_IG -.->|on error| WF_ERR
    WF_ERR --> Gmail

    %% Estilos
    classDef wf fill:#e0f2fe,stroke:#0369a1,stroke-width:1px,color:#0c4a6e
    classDef store fill:#fef3c7,stroke:#b45309,stroke-width:1px,color:#78350f
    classDef saas fill:#f5f3ff,stroke:#6d28d9,stroke-width:1px,color:#4c1d95
    classDef external fill:#dcfce7,stroke:#15803d,stroke-width:1px,color:#14532d
```

## Cómo se relacionan los dominios

1. **Asistente Isabel** es un sistema vivo en tiempo real: cada turno de conversación dispara el orquestador del widget Chatwoot, que combina memoria de Redis, RAG (PGVector poblado por el workflow de ingestión) y dos sub-workflows como tools de Cal.com.
2. **Pipeline RRSS LinkedIn**, **Video** e **Instagram Reels** comparten Airtable como cola de trabajo. El editor humano dispara cada workflow desde botones de Airtable. La imagen LK pasa por un paso de aprobación humana (estado `pendiente_aprobacion` → botón "Publicar"); el vídeo LK y los Reels IG son fire-and-forget (publican directo). Instagram es un pipeline paralelo al de vídeo LK porque exige formato vertical 9:16 con subtítulos quemados y publicación vía Instagram Graph API.
3. **Error Notifier Global** está configurado como `settings.errorWorkflow` en los otros 11 workflows. Cualquier fallo (HTTP 5xx, credencial caducada, throw en Code node) dispara un email rojo al email operativo con stack + link a la ejecución.
4. **Redis y Postgres** son almacenes compartidos. Redis es operativo (buffer, memoria conversacional, idempotencia, rate-limit). Postgres es analítico/persistente (`documents` para RAG, `agent_turn_logs` para auditoría).
