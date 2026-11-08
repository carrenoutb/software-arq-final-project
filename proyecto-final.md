# Proyecto Final — Agregar préstamo de laptops a `biblioteca-api`

*Arquitectura de Software · Semana 15 · Individual o en parejas (1–2 estudiantes)*

---

## 1. El reto

> La biblioteca comenzará a prestar **laptops**, además de los libros que ya presta. Tu trabajo como arquitecto(a): **evolucionar la arquitectura de `biblioteca-api`** para soportar el préstamo de laptops **sin romper nada de lo que ya funciona**.

Este es tu proyecto final del curso: no hay receta única. **Tú identificas las opciones de diseño, eliges una con criterio arquitectónico y la implementas** — y defiendes tu decisión frente a la clase.

Para desarrollarlo, sigue la **guía de ayuda** ([`guia-ayuda-proyecto-final.md`](guia-ayuda-proyecto-final.md)): te acompaña fase por fase con el proceso, los criterios para saber que vas bien y un ejemplo resuelto en cada etapa ([`proyecto-ejemplo/`](proyecto-ejemplo/)). La guía no te da la solución de las laptops: te muestra **cómo llegar** a tu propia decisión.

**Modalidad:** individual o en **parejas** (1–2 estudiantes). En parejas, ambos deben poder explicar **cada decisión**; la defensa oral será individual sobre el trabajo del equipo.

---

## 2. Contexto del sistema

Trabajas sobre el proyecto que ya conoces de todo el curso:

- **`biblioteca-api`** — API REST en **Go + Gin + MySQL**, publicada en GitHub: **[github.com/carrenoutb/biblioteca-api](https://github.com/carrenoutb/biblioteca-api)** (repo público).
- Arquitectura **por capas**: `handler → service → repository` sobre MySQL, con autenticación **JWT** y roles (`student` / `admin`).
- **Dockerizado**: `docker-compose.yml` levanta `mysql` (:3306), `api` (:8080) y `adminer` (:8081).
- Endpoints actuales: autenticación (`/api/auth/*`), libros (`/api/books*`) y préstamos de libros (`/api/loans*`), más `/health` y `/ping`.

Has visto este sistema desde tres ángulos este semestre: lo **levantaste** con Docker (semana 10), lo **diagramaste** con UML (semanas 12–13) y le **agregaste** tu primer endpoint `/ping` (semana 14). Ahora le agregas un módulo completo.

---

## 3. Fase 1 — Análisis del problema (identifica las opciones)

**Objetivo:** entender *dónde* está acoplado el préstamo actual a "libro" y **descubrir las posibles rutas de diseño** antes de decidir.

> 📎 **Ejemplo del resultado:** [`proyecto-ejemplo/fase-1-analisis-del-problema.md`](proyecto-ejemplo/fase-1-analisis-del-problema.md) — así se ve esta fase resuelta en un proyecto de ejemplo (con la feature dummy de reservas). No es la solución de las laptops: es el **modelo del proceso**.

Empieza por **leer** el código real (no asumas nada):

- `migrations/001_init.sql` — mira la tabla `loans`: la columna `book_id` es una **llave foránea NOT NULL** hacia `books`.
- `internal/models/loan.go` — `Loan.BookID`, `LoanRequest.BookID`, y `LoanDetail` que hace JOIN con `books`.
- `internal/service/loan_service.go` — `Borrow()` valida `available_copies`, rechaza préstamos duplicados, calcula el vencimiento (14 días) y decrementa copias.
- `internal/handler/routes.go` — cómo están registradas las rutas de préstamo y qué middleware protegen.

**Preguntas guía para tu análisis (responde en tu informe):**

1. *¿Qué pasaría si la biblioteca quisiera prestar mañana proyectores, salas o tablets?* ¿Cuánto costaría cada vez agregar un "recurso" nuevo con el diseño actual?
2. *¿Dónde exactamente está acoplado el préstamo a "libro"?* Lista los puntos del código (tabla, modelo, request, servicio, rutas) que asumen que se presta un libro.
3. *¿Cuáles son las formas de romper ese acoplamiento?* Describe al menos **2–3 opciones de diseño distintas** para modelar el préstamo de laptops. Para cada una, responde:
   - ¿Qué **modifica** de la base de datos existente?
   - ¿Qué **duplica** y qué **reutiliza** del módulo de libros?
   - ¿Qué pasa **a futuro** si se agrega otro tipo de recurso?
   - ¿Cuánto **riesgo** tiene sobre lo que ya funciona?

> 💡 Piensa en términos del curso: acoplamiento y cohesión (semana 3), estilos arquitectónicos (semanas 4–7), extensibilidad como atributo de calidad (semana 10) y la regla de oro — *"una decisión cuesta más barata de cambiar hoy que en dos años"*.

**Entregable de la Fase 1 (va en tu documentación):** el análisis con las opciones identificadas y una **tabla comparativa de trade-offs** (opción · qué modifica · qué duplica · costo de agregar otro recurso · riesgo).

---

## 4. Fase 2 — Decisión con ADR

**Objetivo:** elegir una opción y documentarla como decisión arquitectónica, comparándola con las otras.

> 📎 **Ejemplo del resultado:** [`proyecto-ejemplo/fase-2-adr-decision.md`](proyecto-ejemplo/fase-2-adr-decision.md) — así se ve un ADR-001 resuelto, con alternativas comparadas, criterios explícitos y consecuencias ✅/❌.

Escribe un **ADR** usando la estructura que viste en la semana 12 (Michael Nygard):

| Campo | Qué debe contener |
|---|---|
| **Estado** | Aceptado |
| **Contexto** | El problema (la biblioteca presta laptops), las opciones consideradas y por qué existía la decisión |
| **Decisión** | La opción elegida y **por qué sobre las otras** (argumenta con trade-offs, no con preferencias) |
| **Consecuencias** | Lo que se gana ✅ y lo que se pierde ❌, y qué implicó para la base de datos, las capas y el futuro |

**Reglas del ADR:**

- Debes **comparar al menos 2 opciones distintas** dentro del ADR (la elegida vs. las descartadas). *Si solo describes lo que hiciste sin alternativas, no es una decisión arquitectónica: es una descripción.*
- Las **opciones de negocio** también se deciden y justifican aquí o en un segundo ADR: ¿cada laptop es un **inventario individual** (por serial) o se presta por **modelo** (copias como libros)? ¿días de préstamo? ¿lo prestan solo estudiantes o también admins?
- Usa el vocabulario del curso: acoplamiento, extensibilidad, simplicidad, riesgo.

> 📌 **Este ADR es el 50% de tu nota.** Es el corazón del proyecto: evalúa tu entendimiento del problema y tu capacidad de argumentar trade-offs — no cuál opción elegiste.

---

## 5. Fase 3 — Implementación

**Objetivo:** materializar tu decisión en el código, siguiendo la arquitectura por capas del proyecto.

> 📎 **Ejemplo del resultado:** [`proyecto-ejemplo/fase-3-implementacion.md`](proyecto-ejemplo/fase-3-implementacion.md) — así se ve la implementación resuelta por capas: migración, modelo, repository, service, handler, rutas y el hook de integración.

### Requisito funcional mínimo (obligatorio)

Con tu decisión implementada, la API debe soportar:

| Funcionalidad | Mínimo requerido |
|---|---|
| **Catálogo de laptops** | Crear laptop (admin) · listar inventario (autenticado) |
| **Prestar laptop** | Endpoint autenticado que registra el préstamo (con su vencimiento) |
| **Devolver laptop** | Endpoint que cierra el préstamo y devuelve la laptop al inventario |
| **Mis préstamos de laptops** | Listar los préstamos de laptop del usuario autenticado |
| **Seguridad** | Mismas reglas de auth y roles que el módulo de libros (JWT) |

### Buenas prácticas evaluadas

- **Nueva migración**: crea un archivo de migración (`migrations/002_*.sql`) en lugar de editar el esquema al vuelo. Las migraciones se ejecutan solas la primera vez (igual que `001`).
- **Respetar las capas**: `handler → service → repository`. No metas SQL en handlers ni reglas de negocio en el repositorio.
- **Código consistente**: sigue los nombres y estilo del módulo de libros (puedes usar Go, nombres en inglés, JSON en camelCase — como el resto del proyecto).
- **No tocar lo que funciona** ❌: el módulo de libros, la autenticación, `/health` y `/ping` deben seguir **exactamente igual de bien**.

> 💡 Si tu decisión fue crear un dominio paralelo, los archivos nuevos siguen el patrón de `book_*`/`loan_*` (repos, servicios, handlers). Si tu decisión fue generalizar, lo que "tocas" de lo existente debe quedar **mejor** que como estaba, y el ADR debe explicar por qué valió la pena el riesgo.

---

## 6. Fase 4 — Verificación (regla de oro: nada puede romperse)

**Objetivo:** demostrar, con evidencia, que todo funciona — lo nuevo **y** lo existente.

> 📎 **Ejemplo del resultado:** [`proyecto-ejemplo/fase-4-verificacion.md`](proyecto-ejemplo/fase-4-verificacion.md) — así se ve la verificación resuelta: checklist de "nada existente se rompe" + casos de error con sus estados HTTP.

Antes de entregar, ejecuta y captura (salida real o captura) **cada** punto del checklist:

- [ ] `docker compose up --build` levanta sin errores
- [ ] `docker compose ps` — los 3 contenedores `Up`, MySQL `(healthy)`
- [ ] `GET /health` → `{"status":"ok"}`
- [ ] `GET /ping` → `{"message":"pong"}`
- [ ] `POST /api/auth/register` y `POST /api/auth/login` funcionan
- [ ] `GET /api/books` (con token) responde el catálogo de libros
- [ ] **Préstamo de libros** sigue funcionando: `POST /api/loans`, `GET /api/loans/me`, `PUT /api/loans/:id/return`
- [ ] `POST /api/laptops` (admin) crea una laptop
- [ ] `GET /api/laptops` (autenticado) lista el inventario
- [ ] **Prestar** una laptop disponible y **devolverla** funciona
- [ ] `GET /api/laptop-loans/me` lista los préstamos de laptop del usuario
- [ ] Las laptops **prestadas no se pueden prestar dos veces** (validación real)

> ⚠️ **Criterio de aprobación:** si algo de lo existente se rompe, el criterio "Código funcionando" baja de inmediato. *"Todo debe seguir funcionando perfectamente"* es parte del contrato del proyecto.

---

## 7. Fase 5 — Diagramas actualizados

**Objetivo:** comunicar tu nueva arquitectura con los diagramas del curso (semana 12–13), en **Mermaid** dentro de tu documentación.

> 📎 **Ejemplo del resultado:** [`proyecto-ejemplo/fase-5-diagramas.md`](proyecto-ejemplo/fase-5-diagramas.md) — así se ven los diagramas resueltos: C4, componentes y secuencias con las validaciones visibles.

Debes entregar, dibujando el sistema **con el módulo de laptops incluido**:

1. **C4 Nivel 1 (Contexto)** — el sistema y sus actores (sin cambiar de tema: una sola pregunta por diagrama ✅).
2. **C4 Nivel 2 (Contenedores)** — los bloques ejecutables (App → API → MySQL) según tu decisión.
3. **Diagrama de componentes** — la estructura interna de la API con el módulo de laptops (comparando o integrando con el de libros según tu decisión).
4. **Diagrama de secuencia** — el flujo **"prestar una laptop"** de punta a punta (request → validaciones → persistencia → liberación del inventario), señalando dónde viven tus validaciones de negocio.

> 💡 Si tu decisión modifica la base de datos, refleja el cambio en los diagramas (¿nueva tabla? ¿tabla generalizada?); el diagrama debe ser **coherente con el ADR y con el código real**.

---

## 8. Entrega del proyecto

### Cómo se entrega

1. **Haz un fork** del repositorio público [github.com/carrenoutb/biblioteca-api](https://github.com/carrenoutb/biblioteca-api) (botón **Fork**) → trabajas en **tu** copia.
2. En el fork debe quedar **el código implementado** (commits con mensajes claros, no un commit gigante al final).
3. Agrega en la raíz de tu fork una carpeta **`proyecto-final/`** con la documentación:
   - `analisis.md` — el análisis de la Fase 1 (opciones + tabla de trade-offs).
   - `adr.md` — el/los ADR(s) de la Fase 2.
   - `diagramas.md` — los diagramas de la Fase 5 (Mermaid).
   - `verificacion.md` — evidencias de la Fase 4 (salidas/capturas por checklist).
4. **Comparte el enlace del fork** por la vía que indique el docente.

### Fecha límite

📅 **Miércoles 25 de noviembre de 2026 · 5:00 pm** — el enlace del fork debe estar compartido a más tardar a esa hora (hora de la última clase del semestre).

---

## 9. Rúbrica de evaluación

| Criterio | Qué se evalúa | Porcentaje |
|---|---|---|
| **ADR / Argumentación** | Entendimiento del problema, opciones identificadas, comparación de trade-offs, calidad del ADR (contexto/decisión/consecuencias) y **defensa oral** coherente con lo entregado | **50%** |
| **Diagramas** | C4 (Contexto + Contenedor), diagrama de componentes con el nuevo módulo y secuencia del flujo de prestar laptop — claros, coherentes entre sí y con el código/ADR | **25%** |
| **Código funcionando** | Funcionalidad completa (catálogo, prestar, devolver, listar, seguridad) **y nada existente roto** (checklist de la Fase 4 cumplido) | **25%** |
| **Total** | | **100%** |

### Desglose orientativo de cada criterio

| Criterio (50%) | % |
|---|---|
| Identifica y analiza al menos 2–3 opciones de diseño (con tabla de trade-offs) | 15% |
| ADR: contexto claro + decisión justificada frente a alternativas + consecuencias | 25% |
| Defensa oral (10 min + preguntas): explica su decisión y responde con argumentos | 10% |

| Diagramas (25%) | % |
|---|---|
| C4 Contexto + Contenedor con el módulo nuevo | 10% |
| Componentes con el módulo de laptops | 8% |
| Secuencia de "prestar una laptop" correcta y coherente | 7% |

| Código (25%) | % |
|---|---|
| Funcionalidad nueva completa y segura (JWT/roles) | 15% |
| Nada existente roto (checklist Fase 4) | 10% |

---

## 10. Defensa oral

Cada estudiante (o ambos miembros de la pareja) presenta **10 minutos + 5 minutos de preguntas** en la **semana 16**:

- Cuenta el problema y por qué existía la decisión (2 min).
- Presenta las opciones consideradas y el trade-off central (3 min).
- Muestra el resultado: diagramas + lo que implementaste (3 min).
- Responde cómo validaste que nada se rompió (2 min).
- + 5 min de preguntas — **aquí se evalúa si la decisión es tuya**, no del código copiado.

> 💡 Prepara respuestas para: *"¿por qué descartaste la opción X?", "¿qué pasaría si mañana prestan proyectores?", "¿qué hubiera pasado si rompías el préstamo de libros?"* — todo lo visto en el curso te sirve de respaldo.