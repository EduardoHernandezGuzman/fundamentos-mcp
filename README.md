# Fundamentos MCP

Repositorio de referencia teórica sobre **Model Context Protocol (MCP)**.

El objetivo es tener una guía compacta y ordenada de los conceptos principales
del protocolo: arquitectura, primitivas, transportes, seguridad, testing,
observabilidad y criterios de producción.

## Contenido

- [Fundamentos teóricos](./FUNDAMENTALS.md): referencia completa del curso,
  organizada en 22 secciones.

## Temas cubiertos

- Qué es MCP y qué problema resuelve.
- Arquitectura: host, client y server.
- Tools, resources y prompts.
- JSON-RPC 2.0 y lifecycle de inicialización.
- Transportes: stdio y Streamable HTTP.
- Context injection, sampling, progress y roots.
- Seguridad, autorización y validación en servidor.
- Elicitation y humano en el circuito.
- Paginación, completion y flujos agentivos.
- Testing, CI, observabilidad y despliegue.
- Principios empresariales para servidores MCP.

## Estructura actual

Por ahora el repositorio se mantiene simple:

```text
.
├── README.md
└── FUNDAMENTALS.md
```

Si el contenido crece, puede dividirse más adelante en una carpeta `docs/` por
módulos o temas. Mientras sea una referencia única, mantenerlo en un documento
principal facilita la lectura y la búsqueda.

## Referencias

- [Especificación MCP](https://modelcontextprotocol.io/specification/latest)
- [SDK Python MCP](https://py.sdk.modelcontextprotocol.io/)
