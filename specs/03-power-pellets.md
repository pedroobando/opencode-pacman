# SPEC 03 — Power pellets y modo frightened

> **Status:** Aprobado
> **Depends on:** SPEC 01, SPEC 02
> **Date:** 2026-08-14
> **Objective:** Añadir cuatro power pellets al laberinto que, al ser comidos por Pac-Man, activan un modo frightened de 600 frames durante el cual los fantasmas se vuelven comestibles, se mueven más lento y al azar, y otorgan puntos escalonados al ser comidos.

## Section 1 — Why this spec exists

SPEC 01 y SPEC 02 construyeron las personalidades de persecución y la secuencia de salida de la pen. Para completar la dinámica arcade clásica falta el contrapeso defensivo: un item que invierta temporalmente la relación de poder y permita a Pac-Man cazar a los fantasmas.

## Scope

**In:**

- Añadir cuatro power pellets al mapa, uno en cada esquina clásica: `(1,3)`, `(26,3)`, `(1,23)`, `(26,23)`.
- Usar el carácter `o` en `MAZE_STR` y asignarle el valor de celda `4` en `parseTile`.
- Pac-Man come un power pellet al pasar por su celda: suma 50 puntos, borra el pellet y activa el modo frightened.
- El modo frightened dura exactamente `POWER_MODE_DURATION = 600` frames.
- Mientras el modo esté activo, todo fantasma fuera de la pen:
  - Se mueve a velocidad `GHOST_SPEED * 0.5`.
  - Elige dirección al azar en cada intersección, sin hacer 180° salvo en callejón sin salida.
  - Se dibuja de color azul sólido `#2121ff`.
- Al colisionar con un fantasma asustado:
  - Pac-Man no pierde vida.
  - Se suman puntos según `ghostsEaten` en la secuencia actual: 200, 400, 800, 1600.
  - El fantasma comido regresa inmediatamente a su posición inicial en la pen.
  - Se respeta la secuencia de salida: si el fantasma era posterior al que está activo, vuelve a `idle = true`; si era el activo, sale cuando le corresponda.
- Al perder una vida se cancela el modo frightened y se resetea `ghostsEaten`.
- Los power pellets no cuentan para `dotsRemaining`; la victoria sigue dependiendo solo de los dots normales.

**Out of scope (para specs futuras):**

- Parpadeo blanco/azul al final del modo frightened.
- Fantasma "comido" con ojos que regresan visualmente a la pen.
- Modos scatter/chase alternados.
- Persistencia entre sesiones.
- Sonidos.

## Data model

### Nuevo valor de celda

```js
// src/js/maze.js
// '#' = pared(1), '.' = dot(2), ' ' = vacío(0), '-' = puerta(3), 'o' = power pellet(4)
```

### Estado del juego

```js
// src/js/game.js — campos agregados a createGame
{
  state: 'start',
  score: 0,
  lives: 3,
  dotsRemaining: dots,   // solo cuenta celdas con valor 2
  powerMode: 0,          // frames restantes de modo frightened; 0 = inactivo
  ghostsEaten: 0,        // fantasmas comidos durante el power pellet actual
  grid,
  pacman,
  ghosts,
}
```

### Constantes nuevas

```js
// src/js/game.js
const POWER_MODE_DURATION = 600; // 10 s @ 60 fps
const FRIGHTENED_SPEED = GHOST_SPEED * 0.5;
const GHOST_EATEN_SCORES = [200, 400, 800, 1600];
const POWER_PELLET_POINTS = 50;
const FRIGHTENED_COLOR = "#2121ff";
```

## Implementation plan

1. **Añadir power pellets al mapa en `src/js/maze.js`**: reemplazar los dots en `(1,3)`, `(26,3)`, `(1,23)` y `(26,23)` por `o`. Extender `parseTile` para que `ch === 'o'` devuelva `4`.
2. **Actualizar `createGame` en `src/js/game.js`**: inicializar `powerMode: 0` y `ghostsEaten: 0`. Asegurar que `dotsRemaining` solo cuente celdas con valor `2`.
3. **Comer power pellet en `movePacman`**: si `grid[p.y][p.x] === 4`, poner la celda en `0`, sumar `POWER_PELLET_POINTS` y activar `powerMode = POWER_MODE_DURATION` con `ghostsEaten = 0`.
4. **Aplicar velocidad frightened en `moveGhost`**: si `game.powerMode > 0` y el fantasma no está en la pen, usar `FRIGHTENED_SPEED` en lugar de `GHOST_SPEED`.
5. **IA aleatoria frightened en `decideGhost`**: si `game.powerMode > 0` y el fantasma no está en la pen, elegir dirección al azar entre las opciones válidas (sin 180° salvo callejón).
6. **Colisión con fantasma asustado en `update`**: si `game.powerMode > 0`, al colisionar sumar `GHOST_EATEN_SCORES[ghostsEaten]` (clamp al último valor si se pasa), incrementar `ghostsEaten`, y devolver el fantasma a su spawn inicial con `inPen = true`, `dir = 'up'`, y recalcular `idle`/`releaseDelay` según su posición en la secuencia fija.
7. **Cancelar modo en `resetPositions`**: poner `game.powerMode = 0` y `game.ghostsEaten = 0`.
8. **Dibujar power pellets en `src/js/render.js`**: círculo de radio 5 en las celdas con valor `4`.
9. **Dibujar fantasmas asustados en `src/js/render.js`**: si `game.powerMode > 0` y el fantasma no está en la pen, usar `FRIGHTENED_COLOR` en lugar de `g.color`.
10. **Probar manualmente**: abrir `src/index.html`, verificar los cuatro pellets, activar el modo, comer fantasmas y confirmar puntuación/respawn.

## Acceptance criteria

- [ ] `MAZE_STR` contiene exactamente cuatro caracteres `o` en las posiciones `(1,3)`, `(26,3)`, `(1,23)` y `(26,23)`.
- [ ] `parseTile('o')` devuelve `4`.
- [ ] Los cuatro power pellets se dibujan como círculos más grandes que los dots normales.
- [ ] Comer un power pellet suma 50 puntos y desaparece el pellet.
- [ ] Tras comer un power pellet, todos los fantasmas fuera de la pen se pintan de azul.
- [ ] Durante el modo frightened, los fantasmas fuera de la pen se mueven a la mitad de velocidad.
- [ ] Durante el modo frightened, los fantasmas fuera de la pen eligen dirección al azar en cada intersección.
- [ ] El modo frightened dura 600 frames (±5 frames por redondeo).
- [ ] Al tocar un fantasma asustado, Pac-Man no pierde vida.
- [ ] El primer fantasma asustado comido suma 200 puntos, el segundo 400, el tercero 800 y el cuarto 1600.
- [ ] El fantasma comido regresa a su posición inicial en la pen y retoma su lugar en la secuencia de salida.
- [ ] Al perder una vida se cancela el modo frightened y el contador de fantasmas comidos vuelve a 0.
- [ ] `dotsRemaining` no incluye power pellets; ganar sigue requiriendo comer todos los dots normales.
- [ ] No aparecen errores en la consola del navegador.

## Decisions

- **Yes:** cuatro power pellets en las esquinas clásicas. Es la disposición más reconocible y simétrica del arcade.
- **Yes:** carácter `o` para power pellet. Es legible y no colisiona con los símbolos existentes.
- **Yes:** valor de celda `4`. Mantiene la secuencia 0/1/2/3 y deja espacio para futuros tiles.
- **Yes:** modo frightened de 600 frames. Coincide con la duración clásica del nivel 1 de Pac-Man.
- **Yes:** velocidad reducida al 50% y dirección aleatoria. Es la mecánica arcade más simple de implementar y la más distinguishable de las personalidades normales.
- **Yes:** puntuación escalonada 200/400/800/1600. Refuerza el incentivo de comer varios fantasmas en un mismo power pellet.
- **Yes:** fantasma comido vuelve a la pen y reinicia su lugar en la secuencia. Mantiene la coherencia con SPEC 02 sin agregar ojos ni animaciones de regreso.
- **No:** parpadeo final azul/blanco. Simplifica el render; se puede agregar en una spec posterior.
- **No:** fantasmas en pen se vean afectados por el modo. Evita ruido visual y lógico en la fase de apertura.
- **No:** power pellet cuenta para `dotsRemaining`. En el arcade los pellets especiales son independientes del progreso del nivel.

## Risks

| Riesgo                                                                                                    | Mitigación                                                                                                       |
| --------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------- |
| Un fantasma asustado en un callejón sin salida queda atrapado.                                            | Se permite el giro de 180° cuando no hay otra opción, igual que en el comportamiento normal.                     |
| La velocidad `GHOST_SPEED * 0.5` puede no alinear a celdas enteras.                                       | El sistema de movimiento fraccionario ya lo soporta; los giros solo se aplican cuando el fantasma está alineado. |
| Comer un fantasma justo cuando cruza el túnel puede dejarlo fuera de lugar.                               | El respawn fuerza las coordenadas iniciales y `inPen = true`, ignorando la posición actual.                      |
| Si Pac-Man come un segundo power pellet mientras el primero aún dura, `ghostsEaten` podría no resetearse. | Al comer cualquier power pellet se resetea `ghostsEaten = 0`, independientemente del estado anterior.            |

## What is **not** in this spec

- Parpadeo final del modo frightened.
- Fantasma "comido" con ojos que regresan a la pen.
- Modos scatter/chase temporizados.
- Persistencia entre sesiones.
- Sonidos o efectos visuales adicionales.

Cada uno, si llega, va en su propia spec.
