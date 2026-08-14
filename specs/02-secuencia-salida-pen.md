# SPEC 02 — Secuencia de salida de la pen: fantasmas quietos y activación escalonada

> **Status:** Approved
> **Depends on:** SPEC 01
> **Date:** 2026-08-14
> **Objective:** Modificar la mecánica de la pen para que los cuatro fantasmas inicien alineados y quietos, y cada uno comience a moverse dentro de la pen solo cuando el fantasma anterior haya salido, manteniendo el orden fijo Blinky → Pinky → Inky → Clyde.

## Section 1 — Why this spec exists

SPEC 01 dejó a los fantasmas moviéndose al azar dentro de la pen mientras esperaban su timer. El comportamiento buscado ahora es más cercano a una secuencia de "despertar": todos empiezan inmóviles, el primero sale, y ese evento activa al siguiente. Esto da una fase de apertura más predecible y visualmente clara.

## Scope

**In:**

- Cambiar las posiciones iniciales de los fantasmas a una fila horizontal consecutiva en `y=14`, `x=11,12,13,14`.
- Orden fijo de salida: Blinky → Pinky → Inky → Clyde.
- Al inicio de la partida (y tras cada vida perdida) los cuatro fantasmas están visibles y completamente quietos.
- Blinky sale inmediatamente: inicia `idle = false` y `releaseDelay = 0`.
- Cuando un fantasma abandona la pen (`inPen` pasa de `true` a `false`), el siguiente en la secuencia comienza a moverse aleatoriamente dentro de la pen.
- Mientras esperan su turno dentro de la pen, los fantasmas activos se mueven al azar sin cruzar la puerta ni hacer giros de 180° salvo callejón sin salida.
- Los tiempos de espera entre salidas siguen siendo `GHOST_RELEASE_INTERVAL = 180` frames.
- Al perder una vida se reinicia la posición, el estado `idle` y los timers, repitiendo la secuencia desde Blinky.

**Out of scope (para specs futuras):**

- Power pellets y modo frightened.
- Modo alternado scatter/chase temporizado.
- Fantasma "comido" con ojos que regresan a la pen.
- Cambios en las personalidades de persecución fuera de la pen.
- Persistencia entre sesiones.
- Animaciones o efectos visuales adicionales al activarse un fantasma.

## Data model

### Estado por fantasma

```js
{
  x: 11, y: 14,        // posición actual (puede ser fraccionaria)
  dir: 'up',           // dirección actual
  speed: 0.1,          // GHOST_SPEED
  kind: 'blinky',      // 'blinky' | 'pinky' | 'inky' | 'clyde'
  color: '#ff0000',    // color arcade
  inPen: true,         // true hasta que cruza y < 12
  releaseDelay: 0,     // frames restantes hasta salir de la pen
  idle: true,          // true mientras está quieto esperando que el anterior salga
}
```

### Spawns actualizados

```js
// src/js/maze.js
const GHOST_STARTS = [
  { x: 11, y: 14, kind: "blinky", color: "#ff0000" }, // izquierda
  { x: 12, y: 14, kind: "pinky", color: "#ffb8ff" }, // centro-izquierda
  { x: 13, y: 14, kind: "inky", color: "#00ffff" }, // centro-derecha
  { x: 14, y: 14, kind: "clyde", color: "#ffb852" }, // derecha
];
```

### Constantes reutilizadas

```js
const GHOST_RELEASE_INTERVAL = 180; // 3 s @ 60 fps
```

## Implementation plan

1. **Actualizar `GHOST_STARTS` en `src/js/maze.js`** a las posiciones `y=14`, `x=11,12,13,14` manteniendo el orden Blinky, Pinky, Inky, Clyde.
2. **Agregar el campo `idle` en `createGame`** (`src/js/game.js`): Pinky, Inky y Clyde inician con `idle: true`; Blinky inicia con `idle: false` para salir de inmediato. Blinky obtiene `releaseDelay = 0`; Pinky, Inky y Clyde obtienen `180`, `360` y `540` respectivamente (sin shuffle).
3. **Congelar fantasmas `idle` dentro de la pen**: en `moveGhost`, si `g.inPen && g.idle`, no decrementar `releaseDelay` ni mover el fantasma; solo retornar.
4. **Movimiento aleatorio dentro de la pen para fantasmas activos**: cuando `g.inPen && !g.idle && g.releaseDelay > 0`, decrementar el timer y elegir dirección al azar entre las válidas, filtrando cualquier dirección que cruce una celda de puerta (`grid[ny][nx] === 3`).
5. **Activar al siguiente fantasma al salir**: al inicio de `moveGhost` guardar `const wasInPen = g.inPen`; después de actualizar la posición, si `wasInPen && !g.inPen`, buscar el siguiente fantasma en `game.ghosts` (por orden de array) que aún tenga `idle: true` y cambiarlo a `false`.
6. **Mantener la salida dirigida a la puerta**: cuando `g.inPen && !g.idle && g.releaseDelay <= 0 && g.y >= 13`, usar `headTowardDoor` para dirigirse a `(13, 12)`; al llegar a la celda de la puerta forzar `dir = 'up'`; al cruzar `y < 12` marcar `inPen = false`.
7. **Actualizar `resetPositions`**: restablecer posiciones iniciales, `dir = 'up'`, `inPen = true` y recalcular `releaseDelay` en orden fijo (`0, 180, 360, 540`). Blinky vuelve a `idle = false`; los demás vuelven a `idle = true`.
8. **Probar manualmente**: abrir `src/index.html`, verificar la fila inicial, la quietud inicial, la salida secuencial y el reinicio tras perder una vida.

## Acceptance criteria

- [ ] Los cuatro fantasmas aparecen en `y=14`, `x=11,12,13,14` al iniciar la partida.
- [ ] Durante los primeros frames, ninguno de los cuatro fantasmas se mueve de su celda inicial.
- [ ] Blinky es el primero en abandonar la pen.
- [ ] Inmediatamente después de que Blinky sale, Pinky comienza a moverse dentro de la pen.
- [ ] Pinky sale de la pen aproximadamente 3 s (±5 frames @ 60 fps) después de que Blinky salió.
- [ ] Inmediatamente después de que Pinky sale, Inky comienza a moverse dentro de la pen.
- [ ] Inky sale aproximadamente 3 s después de Pinky.
- [ ] Inmediatamente después de que Inky sale, Clyde comienza a moverse dentro de la pen.
- [ ] Clyde sale aproximadamente 3 s después de Inky.
- [ ] Un fantasma que aún está `idle` no decrementa su `releaseDelay` ni se mueve.
- [ ] Un fantasma activo dentro de la pen no atraviesa la puerta antes de que su `releaseDelay` llegue a cero.
- [ ] Al perder una vida, los fantasmas vuelven a la fila inicial, quedan `idle` y se repite la secuencia completa.
- [ ] El orden de salida siempre es Blinky → Pinky → Inky → Clyde; no hay shuffle.

## Decisions

- **Yes:** posiciones iniciales consecutivas en `y=14`, `x=11,12,13,14`. Simplifica la validación visual de "todos alineados".
- **Yes:** orden fijo de salida Blinky → Pinky → Inky → Clyde. Elimina la variabilidad de SPEC 01 y hace predecible la fase de apertura.
- **Yes:** campo `idle` booleano por fantasma. Es la forma más simple de separar "quieto esperando activación" de "moviéndose dentro de la pen".
- **No:** shuffle aleatorio de release. Queda descartado en favor del orden fijo.
- **No:** movimiento determinista dentro de la pen. Se mantiene el comportamiento aleatorio de SPEC 01 una vez que el fantasma se activa.
- **No:** ocultar fantasmas hasta que el jugador empiece. Se mantienen visibles desde el primer frame, como en SPEC 01.
- **No:** activación por timer global independiente. Se activa explícitamente cuando el fantasma anterior cruza `y < 12`, lo que evita desfases si un fantasma tarda más en salir.
- **Yes:** mantener `GHOST_RELEASE_INTERVAL = 180`. Conserva la cadencia de SPEC 01.

## Risks

| Riesgo                                                                                    | Mitigación                                                                                                     |
| ----------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------- |
| Si la fila `y=14, x=11..14` no fuera transitable, los fantasmas spawnean sobre pared.     | Confirmado en `MAZE_STR`: la fila 14 tiene espacios en `x=11..16`.                                             |
| Dos fantasmas podrían superponerse dentro de la pen.                                      | Aceptado: al inicio están en celdas adyacentes y el movimiento aleatorio puede hacer que se crucen brevemente. |
| El `releaseDelay` de un fantasma `idle` no debe decrementarse, o saldría antes de tiempo. | El código de `moveGhost` retorna temprano cuando `g.inPen && g.idle`, sin tocar el timer.                      |
| Si Blinky no logra salir, los demás nunca se activan.                                     | Blinky inicia `idle = false` y `releaseDelay = 0`, por lo que nunca espera activación.                         |

## What is **not** in this spec

- Power pellets.
- Modo frightened.
- Modo scatter.
- Animaciones de activación.
- Cambios en las IA de persecución fuera de la pen.
- Persistencia entre sesiones.

Cada uno, si llega, va en su propia spec.
