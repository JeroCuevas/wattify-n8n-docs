# 02 — Pipeline RRSS LinkedIn

Pipeline que cubre la generación de texto, generación de imagen, aprobación humana y publicación de un post en LinkedIn como organización. Orquestado desde Airtable (base RRSS).

## Estado y máquina de transiciones

```mermaid
stateDiagram-v2
    [*] --> sin_texto: Idea en Airtable<br/>(campo idea)
    sin_texto --> con_texto: workflow Generar Texto Post LK
    con_texto --> pendiente_aprobacion: workflow Generar Imagen Post LK
    pendiente_aprobacion --> publicado: workflow Publicar Post LK aprobado
    pendiente_aprobacion --> pendiente_aprobacion: regenerar imagen<br/>(re-dispara Generar Imagen)
    publicado --> [*]
```

## 1. Generar Texto Post LK

```mermaid
flowchart LR
    AT_BTN([Botón Airtable<br/>'Crear Texto LK']):::ext
    WH[Webhook<br/>recibe record_id<br/>+ idea]:::node
    GEMINI[Gemini flash<br/>system prompt:<br/>copywriter B2B Wattify<br/>hook + desarrollo + CTA<br/>3-5 hashtags]:::ext
    UPD[Airtable Update record<br/>Texto Post LK = output]:::store

    AT_BTN --> WH --> GEMINI --> UPD

    classDef node fill:#e0f2fe,stroke:#0369a1,color:#0c4a6e
    classDef store fill:#fef3c7,stroke:#b45309,color:#78350f
    classDef ext fill:#f5f3ff,stroke:#6d28d9,color:#4c1d95
```

**Respuesta inmediata al webhook**: `"¡Texto enviado a procesar! Puedes cerrar esta pestaña y volver a Airtable."`

## 2. Generar Imagen Post LK

Dos pasos LLM en cadena: primero un Gemini flash que actúa de director de arte y redacta el **prompt visual en inglés** a partir del texto del post, y después un modelo de imagen (Nano Banana 2) que genera la imagen.

```mermaid
flowchart TD
    AT_BTN([Botón Airtable<br/>'Generar Imagen LK']):::ext
    WH[Webhook<br/>recibe record_id<br/>+ post_linkedin]:::node
    PROMPT[Message a model1<br/>Gemini flash<br/>director de arte:<br/>prompt visual en EN<br/>texto-en-imagen en ES<br/>entre comillas]:::ext
    NB2[Generate an image<br/>Nano Banana 2<br/>retry 3x · wait 5s]:::ext
    PREP[Prepare Attachment<br/>Payload<br/>buffer binario base64<br/>valida menor de 5MB]:::node
    UPLOAD[HTTP POST<br/>Airtable uploadAttachment<br/>field=Imagen Post LK]:::store
    MARK[Airtable Update<br/>Estado Imagen Post LK<br/>= pendiente_aprobacion]:::store

    AT_BTN --> WH --> PROMPT --> NB2 --> PREP --> UPLOAD --> MARK

    classDef node fill:#e0f2fe,stroke:#0369a1,color:#0c4a6e
    classDef store fill:#fef3c7,stroke:#b45309,color:#78350f
    classDef ext fill:#f5f3ff,stroke:#6d28d9,color:#4c1d95
```

**Respuesta inmediata al webhook**: `"¡Imagen en proceso! En unos segundos la verás en Airtable lista para aprobar."`

Al terminar, el editor humano abre la vista filtrada por `Estado Imagen Post LK = pendiente_aprobacion` y decide si **aprobar** (botón "✅ Publicar en LinkedIn") o **regenerar** (vuelve a disparar este mismo webhook).

## 3. Publicar Post LK aprobado

Workflow disparado solo desde el botón Airtable de aprobación. Valida estado, descarga el attachment, publica en LinkedIn como organización y marca como `publicado`.

```mermaid
flowchart TD
    AT_BTN([Botón Airtable<br/>'✅ Publicar en LinkedIn']):::ext
    WH[Webhook<br/>recibe record_id<br/>responseMode=responseNode]:::node
    GET_REC[Airtable Get Record<br/>fields complete]:::store
    CHECK[Check Status<br/>Code node<br/>valida estado=pendiente_aprobacion<br/>+ attachment + texto]:::node
    IS_VALID{Is Valid?}
    R_INV[Respond Invalid<br/>HTTP 409<br/>error: razón]:::ext

    DL[Download Attachment<br/>HTTP GET attachment.url<br/>responseFormat=file<br/>timeout 30s · retry 3x]:::node
    LK_POST[LinkedIn node<br/>postAs organization<br/>shareMediaCategory IMAGE]:::ext
    PARSE_LK[Parse LK Response<br/>extrae urn<br/>construye URL del post]:::node
    UPD_PUB[Airtable Update<br/>Estado=publicado<br/>Post LK=URL]:::store
    R_OK[Respond Success<br/>HTTP 200<br/>status: published]:::ext

    AT_BTN --> WH --> GET_REC --> CHECK --> IS_VALID
    IS_VALID -->|false| R_INV
    IS_VALID -->|true| DL --> LK_POST --> PARSE_LK --> UPD_PUB --> R_OK

    classDef node fill:#e0f2fe,stroke:#0369a1,color:#0c4a6e
    classDef store fill:#fef3c7,stroke:#b45309,color:#78350f
    classDef ext fill:#f5f3ff,stroke:#6d28d9,color:#4c1d95
```

## Notas operativas

- El estado `pendiente_aprobacion` actúa como **lock pesimista**. Solo se puede publicar desde ese estado; cualquier otro devuelve 409.
- El paso `Parse LK Response` busca el URN del post en múltiples ubicaciones (`r.id`, `r.urn`, `r.activity`, `r.headers['x-restli-id']`, `r.headers['x-linkedin-id']`, `r.body.id`, `r.body.urn`) porque la API de LinkedIn lo devuelve en sitios distintos según versión y endpoint.
- El `Marcar Publicado` tiene `onError: continueRegularOutput` para que aunque Airtable falle al actualizar, el cliente vea la respuesta HTTP 200 (el post ya está publicado, no se puede deshacer).
- El campo `Imagen Post LK` es de tipo attachment de Airtable. La subida usa el endpoint `uploadAttachment` con el body codificado en base64.
- Diferencia con la primera versión del flujo: hasta el 2026-05-26 había un único workflow que generaba imagen **y** publicaba. Se separó en dos para introducir la aprobación humana intermedia.
