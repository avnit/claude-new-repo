# claude-new-repo

<!-- ARCH-DIAGRAM:START -->

## Architecture

> Auto-generated architecture diagram. See [`docs/context-map.md`](docs/context-map.md) for the full context map (core application, containers/cloud, and database connections).

```mermaid
flowchart TD
  User([User / Client])
  UI["Frontend:8080<br/>React"]
  App["claude-new-repo<br/>FastAPI + Uvicorn"]
  Deploy["Google Cloud Run"]
  User --> UI
  UI --> App
  App -.deploy.-> Deploy
```

<!-- ARCH-DIAGRAM:END -->
