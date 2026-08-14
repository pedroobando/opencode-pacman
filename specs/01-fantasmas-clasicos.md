# SPEC 01 — Fantasmas clásicos con cuatro personalidades

> **Status:** approved
> **Depends on:** —
> **Date:** 2026-08-14
> **Objective:** Reemplazar los dos fantasmas actuales por cuatro fantasmas con personalidades distintas estilo Pac-Man arcade (Blinky, Pinky, Inky, Clyde), liberándolos desde la pen de forma escalonada.

## Section 1 — Why this spec exists

El juego cuenta hoy con dos fantasmas (`hunter` y `random`) cuya única diferencia real es que uno persigue y otro no. Para acercarse al original arcade se requieren cuatro personalidades reconocibles —el perseguidor directo, el emboscador, el flanqueador y el cobarde— y un mecanismo de release que evite que los cuatro caigan sobre Pac-Man al mismo tiempo.

## Scope

**In:**

- Cuatro fantasmas: `blinky`, `pinky`, `inky`, `clyde`, cada uno con su propia IA.
- `blinky` persigue por distancia Manhattan a Pac-Man (el "agresivo").
- `pinky` apunta a la celda `PINKY_AHEAD` (4) casillas adelante de Pac-Man en su dirección actual, con clamp a los bordes del laberinto.
- `inky` calcula su target duplicando el vector de Blinky hacia Pac-Man: `target = pacman + 2·(pacman − blinky)`.
- `clyde` persigue a Pac-Man si la distancia Manhattan supera `CLYDE_FLEE_DIST` (8); si no, huye a la esquina inferior-izquierda `(1, 29)`.
- Todos spawnan dentro de la pen. Mientras están dentro y su timer no expiró, se mueven al azar (sin giro de 180°).
- `blinky` sale de la pen en el frame inicial. Los otros tres salen en orden aleatorio, cada 3 s (180 frames @ 60 fps) tras el anterior.
- Al cruzar la celda de la puerta (`y === 12`, `x ∈ {13, 14}`), el fantasma fuerza `dir = 'up'`.
- Cuando Pac-Man pierde una vida, los cuatro fantasmas vuelven a la pen y se recalculan los timers de release.

**Out of scope (para specs futuras):**

- Power pellets y modo frightened.
- Modo alternado scatter/chase temporizado.
- Fantasma "comido" con ojos que regresan a la pen.
- Animaciones distintas por fantasma (más allá del color arcade).
- Sincronización entre `inky` y `blinky` más allá del target descrito.
- Persistencia entre sesiones.

## Data model

### Estado por fantasma

```js
{
  x: 13, y: 14,        // posición actual (puede ser fraccionaria)
  dir: 'up',           // dirección actual
  speed: 0.1,          // GHOST_SPEED
  kind: 'blinky',      // 'blinky' | 'pinky' | 'inky' | 'clyde'
  color: '#ff0000',    // color arcade
  inPen: true,         // true hasta que sale por la puerta
  releaseDelay: 0,     // frames restantes hasta salir de la pen
}
```

### Spawns

```js
// src/js/maze.js
const GHOST_STARTS = [
  { x: 13, y: 14, kind: "blinky", color: "#ff0000" }, // debajo de la puerta, sale inmediato
  { x: 11, y: 14, kind: "pinky", color: "#ffb8ff" }, // izquierda pen
  { x: 14, y: 14, kind: "inky", color: "#00ffff" }, // derecha puerta
  { x: 16, y: 14, kind: "clyde", color: "#ffb852" }, // derecha pen
];
```

### Constantes nuevas en `src/js/game.js`

```js
const GHOST_RELEASE_INTERVAL = 180; // 3 s @ 60 fps
const CLYDE_FLEE_DIST = 8;
const PINKY_AHEAD = 4;
const CLYDE_SCATTER = { x: 1, y: 29 };
```

### Constantes eliminadas

- Los `kind: 'hunter'` y `kind: 'random'` dejan de ser válidos.

## Implementation plan

1. **Ampliar `GHOST_STARTS` en `src/js/maze.js`** con los cuatro fantasmas y su `color`. El HTML no necesita tocarse.
2. **Eliminar `GHOST_COLORS` de `src/js/render.js`** y dibujar el color desde `g.color` en `drawGhost`. La lista de 4 colores ya estaba alineada con el orden nuevo.
3. **Agregar en `createGame`** los campos `color`, `inPen`, `releaseDelay`. Inicializar: `blinky.releaseDelay = 0`; los otros tres con valores `180`, `360`, `540` asignados en orden aleatorio.
4. **Implementar `decideBlinky`** (target = pacman). Reemplaza al `hunter` actual.
5. **Implementar `decidePinky`** (target = celda 4 adelante de Pac-Man en su `dir`, clamp a bordes).
6. **Implementar `decideInky`** (target = `pacman + 2·(pacman − blinky)`, redondeado).
7. **Implementar `decideClyde`** (target = pacman si `|dx|+|dy| > 8`, si no `CLYDE_SCATTER`).
8. **Implementar el modo `inPen`** en `moveGhost`:
   - `releaseDelay > 0`: decrementar, elegir dirección al azar entre válidas (sin 180°).
   - `releaseDelay <= 0` y `y >= 13`: elegir dirección que minimice Manhattan a `(13, 12)`; en empates, respetar el orden de `DIRS`.
   - En la celda de la puerta (`y === 12`, `x ∈ {13, 14}`): forzar `dir = 'up'`.
   - Al pasar a `y < 12`: marcar `inPen = false`.
9. **Refactorizar `decideGhost`** como dispatcher: si `g.inPen`, delega al manejo de pen; si no, llama a `decideBlinky/Pinky/Inky/Clyde` según `g.kind`.
10. **Actualizar `resetPositions`**: resetear `inPen = true` y recalcular `releaseDelay` (blinky = 0, otros shuffleados).
11. **Probar manualmente**: abrir `src/index.html`, verificar los cuatro fantasmas, el release escalonado y el reset post-vida.

## Acceptance criteria

- [ ] `GHOST_STARTS` tiene exactamente cuatro entradas con `kind ∈ {blinky, pinky, inky, clyde}`.
- [ ] Al iniciar una partida, los cuatro fantasmas aparecen dentro de la pen.
- [ ] `blinky` es visible fuera de la pen en el primer frame de juego.
- [ ] El segundo fantasma en salir lo hace 3 s (±5 frames) después de `blinky`.
- [ ] El tercer y cuarto fantasma salen en orden aleatorio, también separados 3 s.
- [ ] Cada fantasma exhibe una ruta visiblemente distinta al perseguir a Pac-Man (no todos van en línea recta).
- [ ] Cuando Pac-Man pierde una vida, los cuatro fantasmas regresan a sus posiciones iniciales en la pen y vuelve a verse el ciclo de release.
- [ ] Mientras un fantasma está en la pen y su timer no expiró, no atraviesa la puerta.
- [ ] Cada fantasma lleva el color arcade: blinky rojo, pinky rosa, inky cian, clyde naranja.
- [ ] No quedan referencias en el código a `kind: 'hunter'` ni `kind: 'random'`.

## Decisions

- **Yes:** personalidades del Pac-Man arcade clásico. Es la referencia más reconocible y mejor balanceada, y satisface el requisito de "uno agresivo" con `blinky`.
- **No:** modos scatter/chase alternados. Agregar temporización global scatter/chase duplica la complejidad sin que el usuario lo pidiera.
- **No:** frightened mode. Queda para una spec futura centrada en power pellets.
- **Yes:** `inPen` boolean + `releaseDelay` counter. Es la representación más simple y testeable del estado de pen.
- **Yes:** timer en frames, no en milisegundos. El loop ya cuenta frames; mezclar `requestAnimationFrame` con `Date.now()` introduce drift.
- **Yes:** mantener todo en `src/js/game.js`. El archivo crecerá a ~300 líneas, todavía manejable; extraer a `ghost_ai.js` se puede hacer en una spec de refactor si hace falta.
- **Yes:** random shuffle del orden de release entre pinky/inky/clyde. Da variedad sin requerir estado extra.
- **No:** seed determinista para el shuffle. El juego no necesita reproducibilidad.
- **Yes:** `g.color` por fantasma en lugar de `GHOST_COLORS[]` indexado por posición. Hace que el orden en `GHOST_STARTS` no sea relevante para los colores.
- **Yes:** clamp de Pinky a los bordes del laberinto. Evita targets inválidos sin agregar lógica de wrap.

## Risks

| Riesgo                                                                         | Mitigación                                                                                                  |
| ------------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------- |
| El timer de release se desfasa si el navegador baja a <60 fps.                 | El timer cuenta frames, no tiempo real. El efecto visual es levemente más lento, pero el orden se mantiene. |
| El target de `inky` puede caer en una pared o celda inválida.                  | La celda solo se usa como referencia para Manhattan distance; no se valida.                                 |
| Tras un reset, los spawns ya ocupados generan superposiciones durante 1 frame. | Se acepta: el arcade original tiene este mismo flicker.                                                     |
| Si el pen tuviera obstáculos internos, `headTowardDoor` podría bloquearse.     | El pen actual es un rectángulo 6×3 abierto. Confirmado en `MAZE_STR`.                                       |

## What is **not** in this spec

- Power pellets.
- Modo frightened.
- Modo scatter.
- Animaciones distintas por fantasma.
- Persistencia entre sesiones.

Cada uno, si llega, va en su propia spec.
