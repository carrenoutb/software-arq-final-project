# Guía de ayuda — Proyecto Final

*Arquitectura de Software · Semana 15 · Complemento del enunciado (`week-15/proyecto-final.md`)*

---

## Cómo usar esta guía

Esta guía **no te da las respuestas del proyecto**: te da el **proceso** para encontrarlas y saber que vas bien. La decisión arquitectónica es tuya — y es lo que se evalúa.

- Trabaja en tu **fork** de `biblioteca-api` y haz **commits por hito** (no uno gigante al final).
- Antes de cada fase: **lee el código real** (rutas, handlers, servicios, repositorios, migraciones). No memorices ; **verifica**.
- Si una fase te queda "difícil", vuelve a los materiales de las semanas indicadas: son tu respaldo.

> ⚠️ **Regla de oro (repetida a propósito):** si una guía te muestra la solución, no es una guía de arquitectura. Cada fase cierra con un **checklist de salida** — úsalo para decidir si pasas a la siguiente.

---

# 🧭 Fase 1 — Análisis del problema

## Objetivo

Entender **dónde y cómo** el préstamo actual está acoplado a "libro", y descubrir las **posibles opciones de diseño** antes de decidir.

> 📎 **Ejemplo del resultado esperado:** [`proyecto-ejemplo/fase-1-analisis-del-problema.md`](proyecto-ejemplo/fase-1-analisis-del-problema.md) — así se ve esta fase resuelta (con la feature dummy de reservas).
>
> 🛠️ **Herramientas:** ninguna obligatoria. Solo tu editor para leer el repo y Markdown para la tabla de trade-offs (opcional: Adminer en `:8081` para ver el esquema real).

## Conceptos del curso que aplican

- **Acoplamiento y cohesión** (Módulo 3): ¿qué tan pegada está la lógica de préstamo a la entidad libro?
- **Arquitectura por capas** (Módulo 3–4): `handler → service → repository` — ¿en qué capa está cada dependencia?
- **Atributos de calidad** (Módulo 6): extensibilidad, mantenibilidad — ¿qué tan difícil es agregar "el próximo recurso"?

## Pasos guiados

1. **Construye el mapa del código.** Abre tu fork y enumera (con rutas reales) qué archivos componen el módulo de préstamos: rutas, handler, service, repository, model y la tabla en la migración. Verifica cada uno con un vistazo rápido — no lo asumas.

2. **Sigue el flujo completo de un préstamo.** Empezando por la ruta `POST /api/loans`, recorre: route → handler → service → repository → SQL. Anota **qué datos viajan** en cada salto y **dónde se toma cada decisión** (¿quién valida copias? ¿quién calcula la fecha de vencimiento? ¿quién escritura en dos tablas?).

3. **Caza el acoplamiento.** Para cada archivo del paso 2, pregúntale al código:
   - ¿Qué campos asumen que se presta un **libro**? (p. ej., `book_id`, `book_title`, `available_copies`)
   - ¿Qué pasará con cada campo si mañana se presta una **laptop**?
   - ¿Qué línea tendría que cambiar **si no fuera un libro**?

4. **Genera opciones (no juzgues todavía).** Con el acoplamiento a la vista, propón **2 o 3 formas distintas** de modelar el préstamo de laptops. Para cada una, responde sin juicios:
   - ¿Qué **toca** de la base de datos existente?
   - ¿Qué **reutiliza** y qué **duplica**?
   - ¿Qué pasa si mañana se prestan proyectores, salas o tablets?

5. **Dale forma a la tabla de trade-offs.** Usa estas columnas (puedes ajustarlas): *opción · qué modifica · qué duplica · costo de agregar otro recurso · riesgo sobre lo existente · complejidad*. Con la última columna llena, ya tienes el material de tu ADR.

## Señales de que vas bien

- Encontraste **todos** (no uno) los puntos donde el préstamo asume "libro" — tabla, modelo, request, consulta SQL, JOIN.
- Tienes **al menos 2 opciones** genuinamente distintas (no variaciones cosméticas de la misma idea).
- Puedes explicar **con el código a la vista** por qué cada opción cuesta algo distinto.
- Tu tabla de trade-offs tiene casillas **negativas** ("riesgo", "costo") — si todo es positivo, no estás comparando de verdad.

## Errores comunes

- ❌ Quedarse con una sola opción: "hago X porque es lo que se me ocurrió" — no es análisis, es decisión sin contexto.
- ❌ Analizar el README y no el código: el acoplamiento vive en las capas, no en la portada.
- ❌ Confundir **"cómo se hace"** con **"por qué se hace así"**: la Fase 1 es diagnóstico; la implementación es la Fase 3.
- ❌ Copiar la *opción* de un compañero: se nota en la defensa y, además, es el ejercicio que se evalúa.

## ✅ Checklist de salida

- [ ] Listé los archivos reales del módulo de préstamos (con rutas).
- [ ] Recorrí el flujo completo `POST /api/loans` y anoté dónde viajan y se deciden los datos.
- [ ] Identifiqué los puntos de acoplamiento a "libro" (tabla, modelo, request, SQL).
- [ ] Propuse **2–3 opciones** distintas de diseño con su tabla de trade-offs (con costos y riesgos reales).

---

# 🧭 Fase 2 — Decisión con ADR

## Objetivo

Elegir una opción **con criterio** y documentarla en un ADR que compare tu decisión con las alternativas.

> 📎 **Ejemplo del resultado esperado:** [`proyecto-ejemplo/fase-2-adr-decision.md`](proyecto-ejemplo/fase-2-adr-decision.md) — así se ve un ADR-001 bien argumentado (incluye el mini-ADR de un caso de borde).
>
> 🛠️ **Herramientas:** ninguna obligatoria. Solo Markdown; la estructura del ADR (Estado/Contexto/Decisión/Consecuencias) ya la tienes como plantilla en el enunciado.

## Conceptos del curso que aplican

- **ADR** (Módulo 5, semana 12): Estado · Contexto · Decisión · Consecuencias — el formato te ordena el razonamiento.
- **Trade-offs** (Módulo 6): toda decisión arquitectónica gana algo y pierde algo. Si no encuentras la pérdida, no buscaste bien.

## Pasos guiados

1. **Define tus criterios de comparación primero.** Antes de elegir, decide *qué pesa* para este proyecto: ¿extensibilidad futura? ¿simplicidad de implementación? ¿riesgo sobre lo que ya funciona? Escríbelos: te servirán para justificar cada punto del ADR.

2. **Redacta el Contexto.** Debe permitir que alguien que nunca vio este curso entienda: el problema (la biblioteca presta laptops), qué opciones consideraste y por qué la decisión existía. Si eliges ahora y escribes el contexto después, queda un vacío: el contexto se escribe **con las opciones a la vista**.

3. **Escribe la Decisión** en una oración clara y verificable (no "creo que…" sino "se implementó X mediante Y").

4. **Sé honesto con las Consecuencias.** Lista lo que ganas ✅ **y** lo que pierdes ❌. Conecta cada consecuencia con tus criterios del paso 1.

5. **Pregunta final antes de cerrar:** *¿esta decisión seguirá teniendo sentido si la biblioteca presta proyectores el próximo semestre?* No necesitas que la respuesta sea "sí" — necesitas **saber qué responderías**.

## Señales de que vas bien

- El ADR compara tu opción **con al menos otra** — si solo describiste lo que hiciste, no es un ADR.
- Las consecuencias negativas existen y no son decorativas ("más código", "más riesgo") — explican **por qué aun así** elegiste.
- Cualquier miembro de tu pareja puede defender el ADR completo solo, sin mirar.

## Errores comunes

- ❌ ADR sin alternativas ("decidimos X" y nada más): pierde la mitad del criterio.
- ❌ Consecuencias todas positivas: pierde credibilidad ante la rúbrica de trade-offs.
- ❌ Cambiar de decisión a mitad de la implementación **sin actualizar el ADR**: la documentación debe reflejar la realidad.
- ❌ Decidir "lo que hace el compañero del lado": la defensa oral pregunta a cada uno.

## ✅ Checklist de salida

- [ ] Definí criterios de comparación (extensibilidad, riesgo, simplicidad…).
- [ ] El ADR tiene Contexto que cualquier persona entiende sin este curso.
- [ ] El ADR **compara ≥2 opciones** y justifica la elegida sobre las demás.
- [ ] Las Consecuencias incluyen lo ganado y lo perdido, conectadas con mis criterios.

---

# 🧭 Fase 3 — Implementación

## Objetivo

Llevar tu decisión al código **respetando la arquitectura por capas** del proyecto y el requisito funcional mínimo del enunciado.

> 📎 **Ejemplo del resultado esperado:** [`proyecto-ejemplo/fase-3-implementacion.md`](proyecto-ejemplo/fase-3-implementacion.md) — así se ve la implementación por capas de la feature de ejemplo (migración, modelo, repository, service, handler, rutas y hook).
>
> 🛠️ **Herramientas:** **Docker** (obligatorio: `docker compose up --build`), **REST Client** en VS Code (probar endpoints, tutorial en el README) y **Go** para `go build ./...` / `go vet ./...`. Opcional: **Adminer** en `:8081` para verificar la BD.

## Conceptos del curso que aplican

- **Patrón por capas** (Módulo 3–4): `handler → service → repository` — cada capa con su única responsabilidad.
- **Migraciones** (que ya conoces): cómo `001_init.sql` crea el esquema al arrancar por primera vez.
- **Registro de rutas y middleware** (taller de `/ping`, semana 14): cómo se agrega una ruta `router.GET(...)` y cómo se protege un grupo.
- **Ciclo editar → build → up → probar** (semana 14): el rebuild es obligatorio para que el contenedor vea tu código.

## Pasos guiados (patrones, no soluciones)

1. **Prepara el terreno.** En tu fork: `docker compose up --build` y verifica que **hoy** todo funciona (health, ping, login, books, préstamos). Si algo ya falla, ese es tu problema de partida — anótalo. Haz un commit inicial "estado base verificado".

2. **Diseña la migración nueva.** Crea `migrations/002_*.sql` siguiendo el **estilo de `001`** (mismo encabezado, `CREATE TABLE IF NOT EXISTS`, tipos coherentes). Pregúntate: *¿qué debe ser único? ¿qué debe ser obligatorio?* — las restricciones al nivel de BD protegen la integridad aunque el código falle. Verifica que se ejecute solita al reiniciar el contenedor (borra el volumen con `down -v` solo si es seguro para tu fork).

3. **Construye de abajo hacia arriba, probando cada capa:**
   - **Modelos** → define la estructura y el request (mira el estilo de `models/book.go`).
   - **Repository** → consultas SQL mínimas (crear, listar, buscar, actualizar estado) — el estilo de `book_repository.go` es tu referencia.
   - **Service** → reglas de negocio y validaciones, *antes* de escribir (¿qué pasa si se presta dos veces? ¿qué pasa al devolver?).
   - **Handler** → recibe la petición, llama al servicio, responde JSON.
   - **Rutas** → registra los endpoints con su middleware (revisa cómo `routes.go` agrupa rutas públicas, protegidas y admin).

4. **Prueba a medida que construyes.** Cada endpoint nuevo se prueba con REST Client (el tutorial del README: petición → respuesta esperada → qué pasó). No esperes a tener "todo listo": un endpoint a la vez.

5. **Cierra con la regla de oro:** vuelve a probar **todo lo existente** (health, ping, auth, books, préstamos de libros). Si algo dejó de funcionar, es un bug tuyo, no un "detalle".

## Señales de que vas bien

- `go build ./...` y `go vet ./...` pasan sin errores.
- Cada capa está en su archivo (no SQL en handlers, no reglas de negocio en repos).
- El flujo de prestar laptop **valida de verdad**: prestar dos veces la misma laptop debe fallar.
- `git log` muestra commits por hito, no un único "proyecto final" al cierre.

## Errores comunes

- ❌ Olvidar el rebuild: el contenedor responde "viejo" y crees que tu código no funciona.
- ❌ Copiar el módulo de libros y cambiar nombres sin entenderlo (se nota en la defensa).
- ❌ Exponer rutas **sin middleware** (¡cualquiera podría prestar!): revisa los grupos en `routes.go`.
- ❌ Modificar `books` o `loans` "un poquito" para entrarle a la fuerza: si tu decisión era no tocarlos, no los toques.

## ✅ Checklist de salida (del código)

- [ ] Creé la migración `002_*.sql` y se ejecuta al arrancar el contenedor.
- [ ] Modelos, repository, service y handler de laptops siguen el estilo del proyecto.
- [ ] Rutas registradas con su middleware (auth y roles correctos).
- [ ] `go build ./...` y `go vet ./...` sin errores.
- [ ] Probé cada endpoint nuevo con respuesta esperada y casos de error (prestar dos veces, devolver sin préstamo).
- [ ] **Lo existente sigue igual** (verificaré completo en la Fase 4).

---

# 🧭 Fase 4 — Verificación

## Objetivo

Demostrar con **evidencia real** que todo funciona: lo nuevo y lo existente.

> 📎 **Ejemplo del resultado esperado:** [`proyecto-ejemplo/fase-4-verificacion.md`](proyecto-ejemplo/fase-4-verificacion.md) — así se ve la verificación resuelta: checklist de "nada existente se rompe" + casos de error con sus estados HTTP.
>
> 🛠️ **Herramientas:** **Docker** (servicios corriendo), **REST Client** en VS Code (o Swagger UI / `curl`) para ejecutar las peticiones, y **Adminer** en `:8081` para comprobar estados en la BD.

## Conceptos del curso que aplican

- **Verificación vs. prueba** (si lo viste): no es "presionar el botón y ver si responde" — es comprobar **casos esperados y casos de error**.
- **Checklist del enunciado** (Fase 4): la lista que debes ejecutar, no solo leer.

## Pasos guiados

1. **Ejecuta el checklist del enunciado en orden** (es la sección 6 de `proyecto-final.md`). No saltes pasos: el orden existe para detectar *qué* rompió *qué*.
2. **Prueba casos de error, no solo el camino feliz:**
   - ¿Prestar una laptop ya prestada? Debe **fallar**.
   - ¿Devolver una laptop sin préstamo activo? Debe **fallar** con mensaje claro.
   - ¿Usar un endpoint de laptop sin token? Debe dar **401**.
3. **Captura evidencia real**: salida de terminal o captura, **por punto**. Una captura vale más que "lo probé y funcionó".
4. **Si algo se rompe:** no lo maquilles — arréglalo y vuelve a correr **desde el inicio del checklist**. Un checklist que "pasa" a medias no demuestra nada.

## Señales de que vas bien

- Cada ítem del checklist tiene su salida capturada.
- Los casos de error responden con el **estado HTTP correcto** (401, 400/404/409) y un body explicativo.
- Ejecutaste el checklist **completo** al menos una vez de corrido, sin excepciones.

## Errores comunes

- ❌ Capturar solo el caso feliz (prestar una laptop disponible).
- ❌ "Probar" mirando el código en vez de ejecutando las peticiones.
- ❌ Dejar el checklist con puntos sin respuesta — es señal directa al evaluador.

## ✅ Checklist de salida

- [ ] Checklist completo del enunciado ejecutado de corrido.
- [ ] Casos de error probados (duplicado, devolución inválida, sin token).
- [ ] Evidencias capturadas por punto (salidas/capturas reales).
- [ ] Nada existente roto (sección "lo existente" del checklist al 100%).

---

# 🧭 Fase 5 — Diagramas

## Objetivo

Comunicar tu nueva arquitectura con los diagramas del curso, **coherentes entre sí y con tu código/ADR**.

> 📎 **Ejemplo del resultado esperado:** [`proyecto-ejemplo/fase-5-diagramas.md`](proyecto-ejemplo/fase-5-diagramas.md) — así se ven los diagramas de la feature de ejemplo: C4, componentes y secuencias con sus validaciones visibles.
>
> 🛠️ **Herramienta:** **Mermaid (obligatorio)** — se renderiza solo en archivos `.md` dentro de GitHub (tu fork). Sin diagramas Mermaid, esta fase no se puede entregar como pide el enunciado.

## Conceptos del curso que aplican

- **C4** (semana 12): una pregunta por diagrama, cada nivel cabe en una hoja.
- **UML aplicado** (semana 13): secuencia, componentes, despliegue — solo los que comunican tu decisión.
- **Mermaid**: el formato que se renderiza en GitHub (donde está tu fork).

## Pasos guiados

1. **Empieza por el C4 Contexto** (¿quién usa el sistema y con qué? — actores externos, sin detalles técnicos). Después Contenedores (los bloques ejecutables). No mezcles niveles en un solo diagrama.
2. **Dibuja el diagrama de componentes** de la API **con tu módulo de laptops** (explícito cómo se relaciona con el módulo de libros: separado, compartido o integrado — según tu decisión del ADR).
3. **Dibuja la secuencia de "prestar una laptop"** de punta a punta: petición → middleware → handler → service → validaciones (deben **aparecer en el diagrama**) → persistencia → respuesta. Si en tu secuencia no se ven las validaciones de negocio, el diagrama no cuenta tu historia.
4. **Cross-check de coherencia:** el diagrama debe ser la *foto* de tu ADR y de tu código. Si el ADR dice "módulo separado" y el diagrama muestra tablas compartidas, hay una desconexión — corrígela antes de entregar.
5. **Verifica que se rendericen:** pega el bloque Mermaid en el archivo del fork y confirma que GitHub lo dibuja (Mermaid se renderiza solo en `.md`).

## Señales de que vas bien

- Cada diagrama responde **una** pregunta (la regla de oro de la semana 12).
- Los 4 diagramas cuentan **la misma historia** (ADR ↔ código ↔ diagrama).
- La secuencia muestra **dónde y cuándo** fallan las validaciones (no solo el camino feliz).

## Errores comunes

- ❌ El "diagrama de todos los diagramas": 50 cajas mezclando niveles y audiencias.
- ❌ Diagramas lindos pero **incompatibles con el código** (ej. tablas que no existen).
- ❌ Copiar el diagrama de componentes de la semana 13 sin el módulo de laptops.

## ✅ Checklist de salida

- [ ] C4 Contexto y Contenedor con el módulo nuevo.
- [ ] Componentes mostrando la relación con el módulo de libros según mi decisión.
- [ ] Secuencia de "prestar una laptop" con validaciones visibles.
- [ ] Los diagramas son coherentes con el ADR y el código.
- [ ] Mermaid se renderiza en el fork.

---

# 🎤 Pre-defensa (opcional pero recomendado)

Antes de la semana 16, responde en voz alta (o con tu pareja):

1. "'¿Cuál era el problema? ¿Por qué existía la decisión?" (2 min — sin diapositivas, de memoria).
2. "'¿Qué opciones consideré y por qué descarté las otras?" (la parte más fuerte de tu ADR).
3. "'¿Qué construí y qué diagrama lo muestra?" — abrir el fork y señalar.
4. "'¿Cómo sé que no rompí nada?" — mostrar la evidencia de la Fase 4.
5. "'¿Qué pasaría si mañana prestan proyectores?" — la pregunta que debes poder responder con tu decisión.

> 💡 La defensa evalúa **si la decisión es tuya**: si puedes contestar estas 5 en orden, estás listo(a).

---

# 🧰 Anexo — Herramientas y atajos

| Necesito… | Cómo |
|---|---|
| Levantar el proyecto | `docker compose up --build` (desde tu fork) |
| Ver si algo se rompió | `docker compose ps` · probar `/health`, `/ping` primero |
| Ver los logs de la API | `docker compose logs -f api` |
| Probar endpoints | REST Client en VS Code (tutorial: sección 3.4 c del README) |
| Revisar la BD | Adminer en `http://localhost:8081` (login: servidor `mysql`, credenciales de tu `.env`) |
| Dibujar diagramas | Mermaid en tus `.md` (se renderiza en GitHub) |
| Commits por hito | `git add archivo && git commit -m "mensaje claro"` — frecuente, no al final |
| Reconstruir la BD de mi fork | Solo si es seguro: `docker compose down -v && docker compose up --build` |

> 📌 Recuerda: la **guía es el proceso**, la **decisión es tuya**. Cada fase cierra con un checklist — si lo cumples, vas bien; si no, vuelve a los pasos de esa fase antes de avanzar.