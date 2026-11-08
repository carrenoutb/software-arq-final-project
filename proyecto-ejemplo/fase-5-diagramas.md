# Fase 5 — Diagramas (ejemplo resuelto)

*Proyecto de ejemplo · Feature dummy: **Reservas de libros** en `biblioteca-api`*

> Este documento es el **resultado** de la Fase 5 del proyecto de ejemplo. Muestra todos los diagramas que pide el enunciado (C4, componentes, secuencia), dibujados en **Mermaid**, coherentes con el ADR-001 (decisión: tabla nueva + módulo paralelo a `loans`).

---

## 1. C4 Nivel 1 — Contexto

**Pregunta:** ¿quiénes interactúan con el sistema y qué esperan de él?

```mermaid
flowchart LR
    Est["Estudiante<br/>(persona)"] -->|"consulta libros,<br/>pide prestado,<br/>reserva agotados"| App["App Web de la biblioteca<br/>(sistema)"]
    Adm["Bibliotecario<br/>(persona)"] -->|"gestiona catálogo,<br/>presta y devuelve"| App
    App -->|"REST + JWT"| API["biblioteca-api<br/>(sistema a diagramar)"]
    API -->|"SQL"| DB[("MySQL 8")]
```

**¿Qué muestra?** El sistema nuevo en el centro, sus actores humanos a la izquierda y el almacén (BD) a la derecha. Una sola pregunta; sin cajas de implementación como routers o handlers.

---

## 2. C4 Nivel 2 — Contenedores

**Pregunta:** ¿de qué bloques ejecutables está compuesta la API?

```mermaid
flowchart LR
    App["App React<br/>(SPA en el navegador)"] -->|"HTTP/JSON :8080"| API["biblioteca-api<br/>Go + Gin"]
    API -->|"TCP/SQL :3306"| DB[("MySQL 8<br/>contenedor")]
    ADM["Adminer<br/>:8081"] -. "visualización web" .-> DB
```

**¿Qué muestra?** Los bloques ejecutables reales (mismos de `docker-compose.yml`). Nota: la feature de reservas **no agregó contenedores** — vive dentro de la API y de MySQL.

---

## 3. Diagrama de componentes

**Pregunta:** ¿qué módulos internos tiene la API y dónde vive la feature nueva?

```mermaid
flowchart TB
    subgraph API["API Go + Gin"]
        direction TB
        H["handler/<br/>auth · book · loan · reservation"]
        MW["middleware/<br/>AuthRequired · AdminRequired"]
        SV["service/<br/>auth · book · loan · reservation"]
        RP["repository/<br/>user · book · loan · reservation"]
        MO["models/<br/>user · book · loan · reservation"]
    end
    DB[("MySQL")]

    H --> SV
    H --> MW
    SV --> RP
    RP --> DB
    RP --> MO
    SV --> MO
```

**¿Qué muestra?** La capa `reservation` **paralela** a `loan` dentro de cada paquete (handler/service/repository) — tal como decidió el ADR-001. No hay flechas de `loan` hacia `reservation` en el diagrama estático; la única conexión es el hook en `Return()` (se ve en la secuencia).

---

## 4. Diagrama de secuencia — Reservar un libro agotado

**Pregunta:** ¿qué ocurre, en orden, cuando un estudiante reserva?

```mermaid
sequenceDiagram
    participant App as App React
    participant MW as AuthRequired<br/>(middleware)
    participant H as ReservationHandler.Reserve
    participant SV as ReservationService.Reserve
    participant BR as BookRepository.FindByID
    participant RR as ReservationRepository
    participant DB as MySQL

    App->>MW: POST /api/reservations (Bearer token)
    MW->>H: valida JWT → user_id en contexto
    H->>SV: Reserve(userID, {book_id})
    SV->>BR: FindByID(bookID)
    BR-->>SV: libro
    alt libro tiene copias disponibles
        SV-->>H: ErrBookHasCopies
        H-->>App: 409 {error}
    else el usuario ya tiene reserva activa del libro
        SV->>RR: HasWaiting(userID, bookID)
        RR-->>SV: true
        SV-->>H: ErrAlreadyWaiting
        H-->>App: 409 {error}
    else todo en orden (libro agotado, sin reserva previa)
        SV->>RR: Create(userID, bookID)
        RR->>DB: INSERT INTO reservations (..., status='waiting')
        DB-->>RR: id
        RR-->>SV: reserva waiting
        SV-->>H: reserva creada
        H-->>App: 201 {reserva}
    end
```

**¿Qué muestra?** El flujo real con **las validaciones de negocio visibles** (copias disponibles y reserva duplicada): son los `alt` del diagrama, y corresponden una a una con los errores de `ReservationService`.

---

## 5. Diagrama de secuencia — La fila avanza al devolver

**Pregunta:** ¿cómo se entera la reserva de que hay una copia disponible?

```mermaid
sequenceDiagram
    participant App as App React
    participant LS as LoanService.Return
    participant LR as LoanRepository
    participant RS as ReservationService.AdvanceQueue
    participant RR as ReservationRepository
    participant DB as MySQL

    App->>LS: PUT /api/loans/:id/return (Bearer token)
    LS->>LR: FindByID + verificar status
    LR->>DB: UPDATE loans SET status='returned'
    LS->>RS: AdvanceQueue(bookID)
    RS->>RR: FulfillOldest(bookID)
    RR->>DB: UPDATE reservations SET status='fulfilled'<br/>WHERE id = (más antigua waiting)
    RR-->>RS: ok
    RS-->>LS: ok
    LS-->>App: 200 {message: "libro devuelto"}
```

**¿Qué muestra?** El **único punto de integración** con el módulo existente: al devolver, `LoanService` llama a una operación semántica (`AdvanceQueue`) sin conocer el esquema de reservas. Este diagrama justifica el hook que se implementó en la Fase 3.

---

## 6. Checklist de coherencia

- [ ] Contexto y Contenedores responden cada uno **una** pregunta.
- [ ] Componentes reflejan la decisión del ADR (módulo paralelo).
- [ ] Las secuencias muestran las **validaciones** (no solo el camino feliz).
- [ ] Los diagramas son **coherentes con el código** real (mismos nombres y endpoints).
- [ ] Todos los bloques Mermaid se renderizan en GitHub (pruébalo en tu fork).

> 📌 **Herramienta:** **Mermaid** (obligatoria) — se renderiza solo en archivos `.md` de GitHub. En tu proyecto, los diagramas van en `proyecto-final/diagramas.md` del fork.