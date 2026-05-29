# Diagramas de las automatizaciones Wattify

Diagramas técnicos del stack de automatización n8n de Wattify. Todos los diagramas están escritos en **Mermaid** embebido, que GitHub renderiza nativamente en la web (no hace falta descargar nada ni instalar herramientas).

## Índice

| Diagrama | Dominio |
|---|---|
| [00 — Vista global del stack](./docs/00-vista-global.md) | Todos los workflows + servicios externos |
| [01 — Agente Isabel](./docs/01-isabel-agente.md) | Chat web Chatwoot + RAG + Cal.com |
| [02 — Pipeline RRSS LinkedIn](./docs/02-rrss-linkedin.md) | Generación texto/imagen + aprobación humana + publicación |
| [03 — Pipeline RRSS Video](./docs/03-rrss-video.md) | Guión + HeyGen + upload chunked a LinkedIn |
| [04 — Pipeline RRSS Instagram](./docs/04-rrss-instagram.md) | Guión + HeyGen 9:16 + publicación vía Instagram Graph API |

## Convenciones visuales

- **Rectángulo redondeado**: nodo de n8n (paso del workflow).
- **Rombo**: nodo `IF` (condicional).
- **Cilindro**: almacén de datos (Redis, Postgres, Airtable).
- **Hexágono / nube**: servicio externo (Gemini, HeyGen, LinkedIn, Cal.com, Chatwoot, Gmail).
- **Flecha continua**: flujo `main` de datos.
- **Flecha discontinua**: conexión `ai_*` (tool / memory / vector store / embedding) o branch paralela.
- **Subgrafo en caja**: agrupa nodos por sub-flujo o por workflow externo.

## Sobre estos diagramas

Esta documentación describe la **arquitectura funcional** de los workflows: qué hacen, cómo se conectan entre sí y con qué servicios externos hablan. Por seguridad, no incluye identificadores internos (IDs de workflows, rutas de webhooks, IDs de base de datos ni emails operativos). Para acceso al detalle de implementación, consulta directamente con el equipo.
