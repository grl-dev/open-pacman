# SPEC 01 — Cuatro fantasmas con personalidades distintas y salida escalonada

> **Status:** Approved
> **Depends on:** —
> **Date:** 2026-08-30
> **Objective:** Ampliar a 4 fantasmas con personalidades de comportamiento diferenciadas (uno persigue agresivamente) y hacer que salgan de la pen de forma escalonada al inicio de cada vida.

## Why this spec exists

Hoy el juego tiene 2 fantasmas, `hunter` (persigue por distancia Manhattan) y `random` (elige al azar entre las opciones válidas). El spec lleva el juego a 4 con los roles clásicos del arcade — uno de ellos claramente agresivo contra PacMan — y añade una cola de salida en la pen para que el jugador no se vea acosado desde el primer frame.

## Scope

**In:**

- 4 fantasmas con `kind` propio: `blinky` (agresivo, persigue directamente), `pinky` (anticipa la posición de PacMan 4 celdas por delante en su dirección), `inky` (flanquea usando la posición de Blinky como referencia), `clyde` (persigue cuando está lejos, se aleja cuando está cerca).
- Salida escalonada de la pen al inicio de la partida y tras perder una vida: Blinky `0 s`, Pinky `2 s`, Inky `5 s`, Clyde `8 s`. Mientras esperan en la pen se mueven arriba/abajo como hasta ahora.
- 4 colores diferenciados en el render: Blinky rojo, Pinky rosa, Inky cian, Clyde naranja.
- Mientras un fantasma está en la pen y no le toca salir, su lógica de IA sigue siendo la actual (rebota arriba/abajo por la geometría de la pen), pero **no avanza** de celda.

**Out of scope (for future specs):**

- Power-pellets y estado de "fantasma asustado / comestible".
- Animación de ojos que vuelven a la pen al ser comido.
- Niveles posteriores con cambios de velocidad o comportamiento.
- IA basada en pathfinding (A*); se sigue eligiendo entre las direcciones válidas en cada intersección, igual que hoy.
- Personalización de dificultad o temporizadores por nivel.

## Data model

`maze.js` — `GHOST_STARTS` pasa de 2 a 4 entradas; las 2 celdas centrales (13,14) y (14,14) se mantienen, y se añaden las 2 laterales superiores dentro de la pen (12,14) y (15,14):

```js
const GHOST_STARTS = [
  { x: 13, y: 14, kind: 'blinky' }, // agresivo, sale a 0s
  { x: 14, y: 14, kind: 'pinky'  }, // anticipa, sale a 2s
  { x: 12, y: 14, kind: 'inky'   }, // flanquea, sale a 5s
  { x: 15, y: 14, kind: 'clyde'  }, // alterna persigue/aleja, sale a 8s
];
```

`game.js` — el objeto `ghost` gana dos campos:

```js
ghosts: GHOST_STARTS.map( ( g, i ) => ( {
  x: g.x, y: g.y, dir: 'up', speed: GHOST_SPEED, kind: g.kind,
  releasedAt: [ 0, 2000, 5000, 8000 ][ i ], // ms desde el inicio de la vida
  out: false, // true cuando ya ha salido de la pen
} ) ),
```

Tabla de personalidades (todas operan sobre la celda `Math.round(x), Math.round(y)` y eligen entre las direcciones válidas en la intersección, igual que `decideGhost` hoy):

| `kind`   | Objetivo calculado                                                          | Notas                                   |
| -------- | --------------------------------------------------------------------------- | --------------------------------------- |
| `blinky` | `Math.round(pacman.x), Math.round(pacman.y)`                                | Mínimo Manhattan.                       |
| `pinky`  | `pacman` adelantado 4 celdas en `pacman.dir`                                | Si `pacman.dir` es `null`, usa su celda.|
| `inky`   | Vector desde `blinky` hasta `pacman` sumado 2 celdas a la posición de `pacman`. | Requiere conocer a Blinky en todo momento. |
| `clyde`  | Si `dist(ghost, pacman) > 8` → celda de PacMan. Si `≤ 8` → celda (2, 30) (esquina inferior izquierda). | El umbral 8 y la celda (2,30) son fijos para este spec. |

Distancia = Manhattan sobre celdas redondeadas. `inky` localiza a Blinky por `game.ghosts.find( g => g.kind === 'blinky' )`.

`render.js` — `GHOST_COLORS` se reordena para casar con el orden de `GHOST_STARTS`:

```js
const GHOST_COLORS = [ '#ff0000', '#ffb8ff', '#00ffff', '#ffb852' ];
//                  blinky       pinky        inky       clyde
```

## Implementation plan

1. **`maze.js`:** reemplazar `GHOST_STARTS` por la versión de 4 entradas anterior. Verificar: `python -m http.server -d src` y abrir — el juego carga con 4 fantasmas visibles en la pen (los 2 nuevos aún sin color asignado caigan al fallback rojo, comportamiento aceptable durante el desarrollo).
2. **`game.js` — campos nuevos:** añadir `releasedAt` y `out` en el `map` de `createGame`. Manual: arrancar partida, los 4 fantasmas aparecen dentro de la pen (sin otra lógica nueva todavía).
3. **`game.js` — gate de salida:** al inicio de `moveGhost`, si `!g.out` y `frameElapsed >= g.releasedAt`, marcar `g.out = true` y forzar `g.x = GHOST_STARTS[blinky].x; g.y = 11` (celda de la puerta, fila 12) y `g.dir = 'up'` para Blinky; los otros 3 hacia arriba para cruzar la puerta. Si `!g.out`, no llamar a `decideGhost` y mantener el rebote actual arriba/abajo. Manual: ver salir a Blinky a los 0 s, Pinky a los 2 s, etc.
4. **`game.js` — personalidades:** sustituir el `if ( g.kind === 'hunter' ) … else …` por una función `targetFor( ghost, game )` que devuelva `{ tx, ty }` según la tabla. Manual: con PacMan quieto en su celda inicial, Blinky converge hacia él; Pinky va hacia 4 celdas arriba (no es intuitivo por el bug histórico del original, pero queda documentado en `Decisions`); Inky describe un arco; Clyde se acerca hasta 8 celdas y luego se va a la esquina.
5. **`game.js` — reset:** `resetPositions` reescribe `x, y, dir` desde `GHOST_STARTS` y resetea `out = false` y `releasedAt` a los valores iniciales. Manual: perder una vida, los 4 vuelven a la pen y Blinky vuelve a salir de inmediato.
6. **`render.js`:** actualizar `GHOST_COLORS` al orden descrito. Manual: los 4 fantasmas se distinguen visualmente desde el primer frame.

## Acceptance criteria

- [ ] `maze.js` declara exactamente 4 entradas en `GHOST_STARTS`, una por cada `kind` (`blinky`, `pinky`, `inky`, `clyde`).
- [ ] Al iniciar una partida, solo Blinky sale de la pen durante los primeros 2 segundos; Pinky no es visible fuera de la pen hasta pasado ese tiempo.
- [ ] Inky no sale de la pen hasta haber pasado 5 segundos desde el inicio de la vida, y Clyde hasta los 8 segundos.
- [ ] Con PacMan quieto en su celda inicial, Blinky reduce su distancia Manhattan a PacMan de forma monótona (no necesariamente cada frame, pero sí en tendencia) hasta colisionar.
- [ ] Clyde, al estar a ≤ 8 celdas de PacMan, se dirige a la celda (2, 30) en lugar de perseguirle.
- [ ] Al perder una vida, los 4 fantasmas regresan a sus celdas iniciales en la pen y Blinky vuelve a salir de inmediato, repitiéndose la cola 0/2/5/8 s.
- [ ] El render muestra los 4 fantasmas con colores rojo, rosa, cian y naranja en ese orden (Blinky, Pinky, Inky, Clyde).
- [ ] La consola del navegador no muestra errores ni warnings nuevos al cargar la página ni al jugar una partida completa.

## Decisions

- **Yes:** 4 personalidades con los nombres y roles del arcade original (`blinky`, `pinky`, `inky`, `clyde`). El usuario los conoce y "uno que persigue agresivamente" encaja con Blinky.
- **Yes:** Salida escalonada con tiempos fijos en segundos (opción A del menú). Determinista, fácil de tunear, no introduce contadores de dots.
- **No:** Tiempos basados en dots comidos. Aplazado por simplicidad para esta primera versión; podría entrar cuando haya niveles.
- **No:** Variar `GHOST_SPEED` por fantasma. Los 4 a 0.1 para no desbalancear el juego simple actual.
- **Yes:** Mientras un fantasma espera en la pen, sigue invocándose su `decideGhost` y queda rebotando arriba/abajo. La celda está limitada por la puerta y las paredes, no hace falta lógica nueva para "congelarlo".
- **Yes:** `inky` se referencia a `blinky` en cada decisión. Mantiene la fórmula del arcade (`2 * pacman + blinky → vector`) y es trivial de localizar.
- **Yes:** Umbral 8 celdas y celda refugio (2, 30) para Clyde. Son los valores del arcade; no se ha justificado cambiarlos en este spec.
- **No:** Pathfinding A* o BFS. Se mantiene la decisión local en cada intersección, igual que la IA actual. Cambiarlo merecería su propio spec.

## Risks

| Risk                                                  | Mitigation                                                                                             |
| ----------------------------------------------------- | ------------------------------------------------------------------------------------------------------ |
| `inky` rompe si Blinky aún no ha salido de la pen     | `targetFor` trata a Blinky como si estuviera en su `GHOST_STARTS` si `blinky.out === false`.           |
| `pinky.dir` es `null` al inicio de la vida            | Si `pacman.dir` es `null`, el target de Pinky es la celda de PacMan (fallback explícito, no NaN).      |
| Cambio de celdas en la pen choca con paredes adyacentes | Las nuevas celdas (12,14) y (15,14) ya existen en `MAZE` como transitables (fila 14 del laberinto).   |
| `releasedAt` se acumula entre vidas si el juego sigue corriendo | `resetPositions` resetea `out` y mantiene el array `releasedAt` original; el contador de tiempo se reinicia porque `startTime` se vuelve a fijar en `startGame`. |

## What is **not** in this spec

- Power-pellets y modo "asustado" (fantasmas azules comestibles).
- Cambios de velocidad por nivel.
- Animación de ojos volviendo a la pen al ser comido.
- Personalización de los umbrales/tiempos desde una UI.
- Cualquier forma de pathfinding global.

Cada uno de esos puntos, si entra, va en su propio spec.
