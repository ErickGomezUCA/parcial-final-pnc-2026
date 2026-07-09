# Erick Bladimir Gomez Hernandez 00300723

## Indicaciones

Recientemente, se utilizó AI para crear un sistema de gestion de una biblioteca, el cual ha generado varios errores, su trabajo es arreglarlo. Dado el siguiente caso de uso, explique y/o resuelva cada problema según se le pida.

---

## Consideraciones

La libreria crea automaticamente un correo con los nombres de la persona

---

## Problemas

### 1. Filtro por autor y género (10%)

QA ha reportado que el endpoint para obtener los libros puede filtrar por **autor** y por **género**, o por cualquiera de los dos de manera individual.

Actualmente:

- Filtrar únicamente por autor funciona correctamente.
- Filtrar únicamente por género funciona correctamente.
- Filtrar por **autor y género al mismo tiempo** provoca que el servidor falle.

**Instrucción:** Explique la causa del problema y resuélvalo.

Explicacion:
1. Los filtros de genre son case sensitive, los filtros que se escriben en minusculas no los logra encontrar con mayusculas (por ejemplo, `novel` falla, pero `NOVEL` funciona porque asi estaba definido el enum de `Genre`). Asi que en este caso, el punto 2 (que genero funciona correctamente) no es del todo cierto. En caso que esto fuese intencionalmente asi, tambien su `status code` de respuesta deberia de haber sido `404`, o `400`, ya que es del lado del cliente, y no con `500` porque el server es el que esta fallando.
2. Los argumentos pasados a la funcion `findByAuthorAndGenre` estan la posicion incorrecta, en vez de ser `author` y `genre`, se estaban pasando `genre` y `author`.
3. El argumento de genre, desde el repositorio, se estaba ingresando como la clase `String`, pero desde el modelo, la entidad se habia definido el enum `Genre`. Asi que se tuvo que corregir para que se pase `genre` como `Genre` (enum) y no como `String` hacia el repositorio.

Solucion:
1. `BookService.java`: Al llamar genre, se asegura que siempre sea con `genre.toUpperCase()` para que sea exacto como esta definido con el enum. 
2. Se cambio la posicion de los argumentos pasados en `findByAuthorAndGenre`, ahora sigue el orden `author` y luego `genre`
3. `BookRepository.java`: Se cambio la clase del argumento `genre`, paso de ser `String` a `Genre` (enum).

---

### 2. Error al volver a prestar un libro (10%)

Un usuario reportó que al pedir prestado el libro **The Selfish Gene**, devolverlo e intentar pedirlo prestado nuevamente, el servidor falla.

**Instrucción:** Explique la causa del problema y resuélvalo.

Explicacion:
1. Para el servicio de movements, al prestar un libro actualiza la cantidad de libros disponibles, si ya no hay libros, entonces actualiza para ese libro que `available = false`, pero, al devolverlo, no regresa que el libro este disponible, a pesar que su `available_count` si increment, si campo `available` se mantiene como `false`. 

Solucion:
1. `MovementeService.java`: Se checkea que el libro no esta disponible
2. En caso de no estarlo, entonces actualizarlo nuevamente a `book.setAvailable(true)`

---

### 3. Cantidad de libros por género (10%)

Existe un endpoint que devuelve la cantidad de libros disponibles por género. Sin embargo, actualmente dicho endpoint falla.

**Instrucción:** Explique la causa del problema y resuélvalo.

Explicacion:
1. En la base de datos, hay un libro que su `genre` es `null`.
2. Entonces, al invocar `book.getGenre().name()`, falla porque no se puede invocar `.name()` para un resultado de tipo `null`.

Solucion:
1. `BookService.java`: Omitir ese libro que tiene `null` en genre.
2. Se hizo saltandolo del conteo con `if (book.getGenre() == null) continue;`

---

### 4. Error al consultar un libro por ID (10%)

Un miembro del equipo de frontend reporta que la siguiente llamada falla:

```http
GET /books?id=ed16ed1e-7017-4697-a08a-d28c09a74acf
```

**Instrucción:** Explique la causa del problema.

Explicacion:
1. El endpoint esta devolviendo todos los libros, ignorando el filtro por ID.
2. Este se debe porque no hay un query param que registre `id`, solo esta tomando en cuenta los filtros `author` y `genre`.

---

### 5. Error al crear un libro (10%)

QA ha reportado que el siguiente payload enviado al endpoint `POST /books` provoca un error:

```json
{
  "title": "Clean Code",
  "author": "Robert C. Martin",
  "genre": "classic",
  "isbn": "978-0132350884",
  "available": true,
  "availableCount": 5
}
```

**Instrucción:** Explique la causa del problema.

Explicacion:
1. Ocurre lo mismo como lo mencionado en el problam 1. La busqueda de los enums son case sensitive, no existe un valor del enum de tipo `classic`, pero si `CLASSIC`.
2. Es decir, con el siguiente cambio en el payload si funciona: `"genre": "CLASSIC", `.
2. Para este caso, se recomendaria agregar un `.toUpperCase` para que pueda coincidir el enum con lo que esta defninido en el enum.

---

### 6. Devolución de libros no prestados (20%)

QA ha reportado que un usuario es capaz de devolver libros que nunca ha solicitado en préstamo.

**Instrucción:**

- Confirme si este comportamiento es realmente posible.
- Si es posible, explique la causa y resuelva el problema.
- Si no es posible, explique por qué, haciendo referencia al código correspondiente.

Confirmacion:
- Si es posible devolver libros que el usuarios no haya pedido prestado

Explicacion:
1. Se han creados dos usuarios, el usuario 1 ha pedido prestado un libro, y el usuario 2 puede devolver ese libro, aunque el no lo haya pedido
2. Esto es debido a que en la funcion `createMovement` nunca se esta revisando que el usuarios que esta devolviendo es el mismo que lo ha pedido prestado.

Solucion:
1. `MovementService.java` + `MovementRepository.java`: Obtener todos los movimientos realizados por ese libro
2. `MovementeService.java`: Al momento de devolver un libro, revisar si el movimiento registrado previo de ese libro coincida con el ID del lector que lo esta devolviendo
3. En caso que el ID del que pidio prestado no sea igual que el ID del que lo esta devolviendo, lanzar la excepcion para evitar que este pueda devolver un libro que no es suyo.
4. En caso que si tengan el mismo ID del lector, entonces se puede devolver el libro.

---