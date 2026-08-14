# AGENTS.md

## Resumen

Juego tipo Pac-Man en **Vanilla JS + HTML + CSS**, sin build step, sin package.json, sin tests automatizados. Se sirve como estáticos desde `src/`. El proyecto sigue Spec Driven Development.

## Cómo correrlo

- Abrir `src/index.html` directamente en el navegador, o servir la carpeta `src/` con cualquier servidor estático (ej. `python3 -m http.server -d src`).
- No hay `npm install`, no hay bundler, no hay TypeScript.

## Estructura y orden de carga (importante)

`src/index.html` carga los `<script>` en este orden estricto. Respetarlo o las globals van a fallar:

1. `js/maze.js` — expone `window.MAZE`, `window.TUNNEL_ROW`, `window.PACMAN_START`, `window.GHOST_STARTS`.
2. `js/game.js` — expone `window.createGame`, `window.update`, `window.DIRS`. Depende de las globales de maze.js.
3. `js/render.js` — expone `window.draw`. Lee `game.grid` (no `MAZE`) para reflejar puntos comidos.
4. `js/main.js` — entrypoint: toma el DOM, registra teclado, arranca `requestAnimationFrame`.

Cada archivo se carga como script clásico (no módulos ES), por eso la comunicación entre ellos es por `window.*`.

## Arquitectura mínima

- **Mapa**: `MAZE_STR` en `src/js/maze.js` es la fuente editable (31 filas de 28 chars: `#` pared, `.` dot, ` ` vacío, `-` puerta). `MAZE` se parsea una vez y es **prístino**: `createGame()` lo cliva por fila para que comer dots no lo destruya.
- **Valores de celda**: `0` vacío, `1` pared, `2` dot, `3` puerta del pen.
  - Pacman es bloqueado por `1` **y** `3`.
  - Fantasmas son bloqueados solo por `1`.
- **Estado del juego**: `state ∈ {'start', 'playing', 'won', 'lost'}` en `game.js`.
- **Loop**: `loop()` en `main.js` incrementa `frame`, llama `update(game)` y `draw(ctx, game, frame)` cada frame via `requestAnimationFrame`.
- **Movimiento**: velocidades fraccionarias (`PACMAN_SPEED = 0.125`, `GHOST_SPEED = 0.1` celdas/frame) — los giros solo se aplican en celdas alineadas (`aligned()` en `game.js`).
- **Túnel horizontal**: `TUNNEL_ROW = 14`, extremos siempre pasables; `wrapTunnel()` reposiciona al cruzar.
- **Fantasmas**: `kind: 'hunter'` persigue por Manhattan, `kind: 'random'` elige al azar; ambos evitan el 180° salvo callejón sin salida.

## Convenciones del código

- Comillas simples y espacios alrededor de paréntesis (estilo WordPress-ish, ver `main.js`). Mantenerlo al editar.
- `const` por defecto; `let` solo donde se reasigna (frame, game).
- Sin comentarios inline nuevos a menos que algo no sea evidente por el código (preferencia del repo: comentarios de cabecera por archivo explicando su rol).
- No introducir frameworks, bundlers, ni package.json sin discutirlo: el repo es deliberadamente minimal.

## Cómo interactuar con el usuario

Idioma: **siempre castellano** para preguntas, opciones y respuestas. Ignorar el idioma del mensaje inicial del usuario — fijarlo a español para esta sección.

Herramienta `question` (`AskUserQuestion`): reservarla para **decisiones complejas o con múltiples opciones** donde hay un tradeoff real que el usuario tiene que resolver (ej.: "en qué consenso caemos", "qué estado de spec elegir", "qué rama base"):

- Una sola tanda por consulta, no encadenar varias llamadas `question` seguidas.
- 2–4 opciones por pregunta, cada una con `label` corto y `description` que explique la implicación.
- Marcar la opción recomendada con `(Recomendado)` en el `label`, y ponerla primera.
- Usar `multiple: true` solo si las opciones son combinables con sentido.

**Preguntas simples / de rápida decisión** (ej.: "¿el archivo va acá o allá?", "¿este nombre está bien?", "¿querés que toque X?"): **no** usar `question`. Hacer la pregunta en texto plano en castellano y seguir la respuesta del usuario, sin interrumpir el flujo con un modal.

## Flujo Spec Driven

El repo usa dos skills en `.agents/skills/` activadas por comandos slash:

- `/spec <descripción>` — diseña una spec en fases (con preguntas) y la guarda en `specs/NN-nombre.md`. Lee `template.md` (en `.agents/skills/spec/`) para la estructura (header con `Status`/`Depends on`/`Date`/`Objective`, Secciones 2 Scope obligatorio, etc.).
- `/spec-impl <NN-spec-name>` — implementa una spec cuyo `Status` sea **Approved** (o equivalente en español: `Aprobado`). Crea la rama con el nombre del spec y avanza por diffs.

**Estados válidos de spec:** `Draft`, `In review`, `Approved`, `Implemented`, `Obsolete` (o equivalentes en español consistentes con specs previas). Si hay specs existentes, matchear su idioma.

Specs viven en `specs/` (carpeta aún no creada en este repo). Convenciones de naming: `NN-nombre-descriptivo.md`.

## Lo que NO hacer

- No convertir a módulos ES ni agregar bundler.
- No mutar `MAZE` directamente; siempre pasar por `createGame()`.
- No agregar lint/test/CI sin pedido explícito.
- No cambiar el orden de los `<script>` en `index.html`.