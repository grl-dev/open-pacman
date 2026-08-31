# SPEC 02 — Salida correcta de los fantasmas de la pen

> **Status:** Approved
> **Depends on:** SPEC 01
> **Date:** 2026-08-31
> **Objective:** Corregir la salida de los 4 fantasmas de la pen para que, cuando les toque su turno (0/2/5/8 s), crucen la puerta y entren al pasillo superior con una dirección prefijada, en lugar de quedarse rebotando dentro.

## Why this spec exists

El SPEC 01 añadió los 4 fantasmas y la cola de salida escalonada, pero arrastra un bug: cuando se cumple `releasedAt`, el código fija `g.x = start.x; g.y = 11; g.dir = 'up'` y acto seguido invoca `decideGhost`, que evalúa Manhattan contra el target de cada `kind` (que tira hacia PacMan o hacia la esquina inferior izquierda). Como PacMan está abajo, casi siempre la opción "más cercana" es volver a bajar, y el fantasma nunca abandona la celda `(x, 11)` — se queda rebotando dentro de la pen indefinidamente, aunque su flag `out` esté a `true`. Resultado: el jugador nunca ve a Pinky, Inky ni Clyde; Blinky a veces sale, otras veces no, según hacia dónde mire PacMan en el instante del cruce.

## Scope

**In:**

- Forzar la salida: cuando `!g.out` pasa a `true`, se fija `g.dir` según una tabla por `kind` y **no** se llama a `decideGhost` en esa misma decisión; el fantasma cruza la puerta y entra al pasillo superior (fila 11) sin desviarse.
- Direcciones de salida prefijadas por fantasma (estilo arcade):
  - `blinky` → `up` (sube recto por el pasillo central).
  - `pinky`  → `left` (gira a la izquierda al entrar a fila 11).
  - `inky`   → `right` (gira a la derecha al entrar a fila 11).
  - `clyde`  → `left` (gira a la izquierda al entrar a fila 11).
- A partir de la celda siguiente a `(x, 11)`, `decideGhost` opera con normalidad: Clyde puede irse a la esquina `(2, 30)` o perseguir a PacMan según la distancia, Inky usa a Blinky como referencia, etc. El spec no toca la IA por `kind`, solo desbloquea la transición.
- Mientras un fantasma está dentro de la pen y no le toca salir, sigue rebotando arriba/abajo como hoy (decisión del SPEC 01, sin cambios).
- Reset de vida: `resetPositions` resetea también la nueva dirección de salida al `dir` inicial de la pen (`up`) y `out = false`, igual que ya hacía con `releasedAt`. La dirección prefijada se vuelve a aplicar en el próximo cruce.

**Out of scope (for future specs):**

- Power-pellets y modo "asustado" (fantasmas azules comestibles).
- Animación de ojos volviendo a la pen al ser comido (los fantasmas no son comibles todavía).
- Niveles posteriores con cambios de velocidad o de los tiempos de la cola.
- IA basada en pathfinding (A*); se mantiene la decisión local en cada intersección.
- Variantes de la dirección de salida por nivel.
- Tiempos de salida basados en dots comidos en lugar de segundos fijos.

## Data model

`maze.js` — `GHOST_STARTS` incorpora la dirección de salida de cada fantasma, conservando el resto del registro:

```js
const GHOST_STARTS = [
  { x: 13, y: 14, kind: 'blinky', exitDir: 'up'    },
  { x: 14, y: 14, kind: 'pinky',  exitDir: 'left'  },
  { x: 12, y: 14, kind: 'inky',   exitDir: 'right' },
  { x: 15, y: 14, kind: 'clyde',  exitDir: 'left'  },
];
```

`game.js` — el campo `exitDir` se propaga al objeto `ghost` desde `createGame`:

```js
ghosts: GHOST_STARTS.map( ( g, i ) => ( {
  x: g.x, y: g.y, dir: 'up', speed: GHOST_SPEED,
  kind: g.kind, exitDir: g.exitDir,
  releasedAt: [ 0, 2000, 5000, 8000 ][ i ],
  out: false,
} ) ),
```

`resetPositions` restaura `dir = 'up'` (la dirección inicial de la pen), `out = false` y `releasedAt` a su valor original; `exitDir` no se muta, se vuelve a aplicar en el siguiente cruce.

`game.js — moveGhost`: el bloque de transición se simplifica. El código actual tiene dos ramas (`if (!g.out) … else …`) que hacen exactamente lo mismo (llaman a `decideGhost`). La forma corregida:

```js
if ( !g.out ) {
  if ( game.frameElapsed >= g.releasedAt ) {
    g.out = true;
    const start = GHOST_STARTS.find( ( s ) => s.kind === g.kind );
    g.x = start.x;
    g.y = 11;          // celda sobre la puerta
    g.dir = g.exitDir; // forzar dirección de salida
  } else {
    decideGhost( game, g ); // rebote en la pen como hasta ahora
  }
} else {
  decideGhost( game, g );   // IA normal fuera de la pen
}
```

Notas clave:

- Cuando el fantasma cruza la puerta en este frame, **no** se ejecuta `decideGhost` en esa misma decisión. Su `dir` ya vale `exitDir` y avanza con esa dirección.
- En el siguiente frame, `aligned(g.y)` puede seguir siendo cierto (porque `11 - speed` sigue ~alineado a una celda) o no; en cualquier caso, la próxima vez que `aligned(g.x) && aligned(g.y)` sea cierto y `g.out === true`, ya entra en la rama `else` y `decideGhost` toma el control.
- `decideGhost` no cambia. Clyde, Inky, Pinky y Blinky siguen aplicando su `targetFor` habitual.
- Ningún cambio en `render.js`. Los 4 colores y `GHOST_COLORS` del SPEC 01 se conservan.

## Implementation plan

1. **`maze.js`:** añadir el campo `exitDir` a cada entrada de `GHOST_STARTS` con la tabla de este spec. Manual: recargar la página, abrir la consola y confirmar que `window.GHOST_STARTS` tiene 4 elementos y cada uno incluye `exitDir` (`blinky: 'up'`, `pinky: 'left'`, `inky: 'right'`, `clyde: 'left'`).
2. **`game.js — createGame`:** propagar `exitDir` desde `GHOST_STARTS` al objeto `ghost`. Manual: arrancar partida y verificar en consola que `game.ghosts[0].exitDir === 'up'`, etc.
3. **`game.js — moveGhost`:** sustituir el bloque `if (!g.out) … else …` actual (que llama a `decideGhost` en ambas ramas) por la versión de tres ramas de la sección anterior. Manual: con PacMan quieto en `(13, 23)`, observar que Blinky sube por el pasillo central a partir del frame 0, sale de la pen y se aleja hacia arriba. Pinky no debe verse fuera de la pen hasta los ~2 s.
4. **`game.js — resetPositions`:** confirmar que sigue reseteando `dir = 'up'`, `out = false`, `releasedAt` original; `exitDir` queda intacto para reaplicarse en el siguiente cruce. Manual: perder una vida deliberadamente (dejar que un fantasma colisione), confirmar que los 4 vuelven a la pen y Blinky vuelve a salir de inmediato con `dir = 'up'`.
5. **`game.js — verificar todos los `kind`:** con PacMan quieto en su celda inicial, confirmar visualmente que Pinky sale a los ~2 s por la puerta y gira a la izquierda; Inky sale a los ~5 s y gira a la derecha; Clyde sale a los ~8 s y gira a la izquierda. Manual: cronometrar con el reloj del navegador.

## Acceptance criteria

- [ ] `window.GHOST_STARTS` contiene exactamente 4 entradas y cada una declara un `exitDir` válido (`'up'`, `'left'` o `'right'`).
- [ ] Al iniciar una partida, Blinky abandona la pen (celda `y ≤ 10`) durante los primeros 2 segundos, con `dir === 'up'` en el momento del cruce.
- [ ] Pinky abandona la pen pasados ~2 s desde el inicio de la vida y, al entrar a fila 11, su `dir` es `'left'`. No vuelve a entrar a la pen.
- [ ] Inky abandona la pen pasados ~5 s y, al entrar a fila 11, su `dir` es `'right'`. No vuelve a entrar a la pen.
- [ ] Clyde abandona la pen pasados ~8 s y, al entrar a fila 11, su `dir` es `'left'`. No vuelve a entrar a la pen.
- [ ] Mientras un fantasma no ha cumplido su `releasedAt`, sigue ejecutándose `decideGhost` y rebota arriba/abajo dentro de la pen (sin cambios respecto a SPEC 01).
- [ ] Al perder una vida, los 4 fantasmas regresan a sus celdas iniciales en la pen, `out` pasa a `false`, `releasedAt` se reinicia a `[0, 2000, 5000, 8000]` y Blinky vuelve a salir de inmediato con la misma mecánica.
- [ ] Con PacMan quieto en `(13, 23)` y Clyde ya fuera, cuando `|gx − 13| + |gy − 23| ≤ 8`, Clyde se dirige a la celda `(2, 30)` y no queda atrapado en la pen.
- [ ] La consola del navegador no muestra errores ni warnings nuevos durante una partida completa (lifecycle: arranque → varias vidas → game over).

## Decisions

- **Yes:** Forzar `dir = exitDir` y saltarse `decideGhost` solo en la decisión del cruce (opción A). Mínimo cambio, mismo comportamiento esperado, sin nuevos estados.
- **No:** Forzar `dir = 'up'` hasta que el fantasma esté fuera de la pen (opción B). Más fiel al arcade, pero introduce un estado "exiting" o reinterpreta `out` y complica `resetPositions`. Aplazado.
- **No:** Teletransportar al pasillo de arriba sin animar (opción C). Pierde la animación clásica de cruzar la puerta; problema de feedback visual.
- **Yes:** Mantener el rebote arriba/abajo en la pen mientras el fantasma espera su turno. Decisión heredada y aprobada del SPEC 01.
- **Yes:** Direcciones de salida arcade: Blinky up, Pinky left, Inky right, Clyde left. Patrón conocido, queda documentado en `GHOST_STARTS` y en este spec.
- **Yes:** Cubrir los 4 `kind` en este spec. El bug afecta a todos por igual (decideGhost devuelve al fantasma a la pen); no tiene sentido arreglar solo a Blinky.
- **No:** Cambiar la IA de Clyde para que evite explícitamente la pen. La fórmula Manhattan sobre la intersección ya lo gestiona una vez corregido el cruce.
- **No:** Añadir `pathfinding` global. La decisión local en cada intersección sigue siendo suficiente para este spec.
- **No:** Tiempos de salida basados en dots comidos. Aplazado (heredado del SPEC 01).

## Risks

| Risk                                                                                | Mitigation                                                                                                                |
| ----------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------- |
| `exitDir` no se restaura al perder una vida y un fantasma sale con dirección vieja | `resetPositions` solo resetea `dir`, `out` y `releasedAt`; `exitDir` queda intacto y se vuelve a aplicar en el próximo cruce. |
| `decideGhost` corrige la dirección **antes** de que el fantasma llegue a fila 11   | Solo se invoca `decideGhost` cuando `g.out === true` y, si el fantasma acaba de cruzar, `g.y = 11` aún no está alineado; entra en la rama correcta en el siguiente `aligned`. |
| `inky` rompe si Blinky aún no ha salido de la pen                                   | Heredado del SPEC 01: `targetFor` usa `GHOST_STARTS[0]` como fallback cuando Blinky está dentro.                          |
| Cambio de filas de la puerta rompe el cruce                                         | La puerta sigue en fila 12 (cols 13-14) y la celda de aterrizaje en fila 11, según `MAZE_STR` actual; no se modifica.      |
| `exitDir = 'left'` para Pinky la choca contra la pared lateral del pasillo          | El pasillo de fila 11 (`'######.##..........##.######'`) tiene huecos transitables; las 4 celdas centrales (12-15) están libres en ambos sentidos. Verificar manualmente. |

## What is **not** in this spec

- Power-pellets y modo "asustado" (fantasmas azules comestibles).
- Animación de ojos volviendo a la pen al ser comido.
- Niveles posteriores con cambios de velocidad o de los tiempos de la cola.
- Personalización de las direcciones de salida desde una UI.
- Cualquier forma de pathfinding global.
- Tiempos de salida basados en dots comidos en lugar de segundos fijos.

Cada uno de esos puntos, si entra, va en su propio spec.
