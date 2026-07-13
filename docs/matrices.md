# Sistema de matrices en Tetris

Este documento explica cómo `game.js` usa matrices para representar el tablero y las piezas, con ejemplos trazados paso a paso.

## 1. Las dos matrices

### Matriz del tablero (`board`)

Dimensión fija `ROWS × COLS` (20 × 10). Cada celda vale:
- `0` → vacía
- `1-7` → color de la pieza que quedó fijada ahí (índice en `COLORS`)

Resuelve dos cosas de forma directa:
- **Colisión**: comprobar `board[y][x]` es una lectura O(1), sin estructuras auxiliares.
- **Limpieza de líneas**: una fila llena se detecta con `board[r].every(v => v !== 0)` y se elimina con `splice` + `unshift` de una fila vacía arriba.

### Matriz de cada pieza (`PIECES`)

Cada pieza se define como una matriz **cuadrada**, rellena con `0` donde no hay bloque:

```js
// T
[[0,3,0],
 [3,3,3],
 [0,0,0]]
```

- `I` es 4×4, `O` es 2×2, el resto (T, S, Z, J, L) son 3×3.
- Que sea cuadrada es intencional: la rotación (`rotateCW`) se apoya en que la matriz no cambie de dimensión al rotar. Si se agregara una pieza no cuadrada, la posición se rompería.

La posición de la pieza (`current.x`, `current.y`) es la esquina **top-left del bounding box**, no del bloque visible — por eso hay ceros de relleno alrededor de la forma real.

## 2. Colisión — `collide(shape, ox, oy)`

Recorre cada celda no-cero de la matriz de la pieza y la traduce a coordenada real del tablero sumando el offset:

```js
for (let r = 0; r < shape.length; r++) {
  for (let c = 0; c < shape[r].length; c++) {
    if (!shape[r][c]) continue; // salta huecos, ni se evalúan
    const nx = ox + c;
    const ny = oy + r;
    console.log(`celda (${r},${c}) valor=${shape[r][c]} → board(${nx},${ny})`);
    if (nx < 0 || nx >= COLS || ny >= ROWS) return true; // fuera de rango
    if (ny >= 0 && board[ny][nx]) return true;           // choque con celda fija
  }
}
```

**Trace real** para la pieza T en `x=3, y=0`:

```
celda (0,1) valor=3 → board(4,0)
celda (1,0) valor=3 → board(3,1)
celda (1,1) valor=3 → board(4,1)
celda (1,2) valor=3 → board(5,1)
```

Los ceros de relleno (`(0,0)`, `(0,2)`, `(2,0)`, `(2,1)`, `(2,2)`) nunca generan un log: el `continue` los descarta antes de tocar el tablero.

## 3. Rotación — `rotateCW(shape)`

Fórmula: `result[c][rows - 1 - r] = shape[r][c]`. Cada celda `(r,c)` de la matriz original pasa a `(c, rows-1-r)` en la nueva: la fila vieja se convierte en columna nueva, invertida.

```js
function rotateCW(shape) {
  const rows = shape.length, cols = shape[0].length;
  const result = Array.from({ length: cols }, () => new Array(rows).fill(0));
  for (let r = 0; r < rows; r++)
    for (let c = 0; c < cols; c++) {
      result[c][rows - 1 - r] = shape[r][c];
      console.log(`shape[${r}][${c}]=${shape[r][c]} → result[${c}][${rows - 1 - r}]`);
    }
  return result;
}
```

**Trace real** rotando la T (rows=3, cols=3):

```
shape[0][0]=0 → result[0][2]
shape[0][1]=3 → result[1][2]
shape[0][2]=0 → result[2][2]
shape[1][0]=3 → result[0][1]
shape[1][1]=3 → result[1][1]
shape[1][2]=3 → result[2][1]
shape[2][0]=0 → result[0][0]
shape[2][1]=0 → result[1][0]
shape[2][2]=0 → result[2][0]
```

Resultado:

```js
result = [
  [0,3,0],
  [0,3,3],
  [0,3,0]
]
```

Visualmente la T rotó 90° en sentido horario.

## 4. Wall kick — `tryRotate()`

Rotar cerca de una pared puede hacer que el nuevo bounding box choque aunque la pieza "debería" poder rotar. `tryRotate()` prueba desplazamientos horizontales antes de descartar el giro:

```js
function tryRotate() {
  const rotated = rotateCW(current.shape);
  const kicks = [0, -1, 1, -2, 2];
  for (const kick of kicks) {
    console.log(`probando kick=${kick}, x=${current.x + kick}`);
    if (!collide(rotated, current.x + kick, current.y)) {
      console.log('rotación aceptada');
      current.shape = rotated;
      current.x += kick;
      return;
    }
    console.log('choque, sigo probando');
  }
}
```

**Ejemplo**: pieza pegada al borde izquierdo (`x=0`).

```
probando kick=0, x=0   → choque (nx=-1, fuera de rango)
probando kick=-1, x=-1 → choque (peor, más afuera todavía)
probando kick=1, x=1   → sin choque
rotación aceptada
```

La pieza se desplaza 1 celda a la derecha para poder completar la rotación.

## 5. Lock — fusión de la matriz de la pieza en el tablero

Recién en `lockPiece()` (vía `merge()`) la matriz de la pieza se copia dentro de la matriz del tablero. Hasta ese momento son estructuras completamente separadas — mover o rotar la pieza nunca toca `board`.

```js
function merge() {
  for (let r = 0; r < current.shape.length; r++)
    for (let c = 0; c < current.shape[r].length; c++)
      if (current.shape[r][c]) {
        board[current.y + r][current.x + c] = current.shape[r][c];
        console.log(`board(${current.x + c},${current.y + r}) = ${current.shape[r][c]}`);
      }
}
```

## Resumen

- La matriz de la pieza siempre usa coordenadas **locales** (empiezan en 0,0); se traducen a coordenadas del tablero sumando `current.x` / `current.y` en cada operación (`collide`, `draw`, `merge`).
- Las dos matrices (pieza y tablero) nunca se mezclan hasta el `lock`.
- El padding con ceros en las piezas es necesario para que la rotación funcione con una fórmula única, pero exige tener presente que la posición `(x,y)` no es la del bloque visible sino la del bounding box completo.
- El valor numérico de cada celda hace doble función: marca "ocupado" (cualquier valor no-cero) y determina el color a dibujar (`COLORS[valor]`). Cualquier cambio al sistema de colores impacta también la lógica de colisión si no se separan ambos conceptos.
