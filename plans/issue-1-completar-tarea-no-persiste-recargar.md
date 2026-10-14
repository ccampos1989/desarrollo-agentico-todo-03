# Bug: completar una tarea no persiste al recargar el navegador

## Encargo y resultado esperado

Issue: https://github.com/ccampos1989/desarrollo-agentico-todo-03/issues/1

- **Observado.** Al pulsar el botón de completar, la tarea aparece como completada. Al recargar la página, vuelve a mostrarse como pendiente.
- **Esperado.** Después de pulsar completar (o reabrir) y recargar, la tarea conserva el estado elegido.

**Criterios de aceptación**

1. Con JavaScript activo, al pulsar completar en una tarea pendiente se persiste `completed = true`. Al recargar, la tarea sigue en `task-completed`.
2. Lo mismo al reabrir: una tarea completada pasa a `completed = false` y lo conserva tras recargar.
3. La petición que envía el JavaScript incluye el `authenticity_token` del formulario y el `_method=PUT`, de modo que pasa el middleware CSRF fuera del entorno de test.
4. Si la petición falla (respuesta no-OK o error de red), la interfaz no se queda mostrando un estado que el servidor no tiene: la página se recarga y muestra el estado real.
5. Sin JavaScript, el formulario sigue funcionando como hoy: PUT mediante `_method`, 303 a `/`.
6. El feedback inmediato introducido en `5fdd6c3` se mantiene: la clase `task-completed`, `aria-pressed`, `aria-label`, `title` y el `.task-status` cambian sin esperar a la red.

## Evidencia y causa raíz

### Hechos observados en el repositorio

- **Ruta del toggle.** `actions/app.go` solo registra `app.PUT("/tasks/{task_id}/toggle", TasksToggle)`. No existe `GET /tasks/{task_id}/toggle`.
- **Formulario del toggle.** En `templates/tasks/index.plush.html`, cada tarea tiene `<form action="/tasks/<id>/toggle" method="POST" class="task-toggle" data-task-toggle>`, con los campos ocultos `authenticity_token` y `_method=PUT`.
- **JavaScript del toggle.** `public/assets/app.js` intercepta el `submit` de los formularios `[data-task-toggle]`:
  1. hace `event.preventDefault()`;
  2. actualiza el DOM;
  3. termina con `fetch(form.action);` (línea 34).

  Sin opciones, `fetch` hace una petición **GET**, sin cuerpo. No envía `_method` ni `authenticity_token`.
- **Por qué la petición no llega a `TasksToggle`.**
  - Buffalo solo aplica el *method override* a peticiones POST que llevan `_method` en el formulario (`buffalo@v1.1.4/method_override.go:15-19`). Un GET sigue siendo GET y no coincide con ninguna ruta.
  - El middleware CSRF (`gobuffalo/middleware/csrf/csrf.go:84`) exige el token en métodos no seguros. Lo lee de la cabecera `X-CSRF-Token`, de `PostFormValue("authenticity_token")` o del multipart (`csrf.go:219-240`).
- **Origen del fallo.** `git log -L` sobre esa línea muestra que `fetch(form.action);` entró en `5fdd6c3 feat(ui): improve task completion feedback`. Antes, el formulario se enviaba de forma nativa.
- **Por qué los tests no lo detectan.**
  - `actions/tasks_test.go` (`Test_TasksToggle_CompletesAndReopens`) prueba el PUT directamente contra el servidor, y pasa.
  - El middleware CSRF se desactiva con `GO_ENV=test` (`csrf.go:50-54`).
  - No hay infraestructura de tests de JavaScript: no existe `package.json` ni un runner de navegador.

### Causa raíz

El JavaScript que da feedback inmediato sustituye el envío nativo por un GET sin cuerpo a una ruta que solo acepta PUT. El servidor nunca ejecuta `TasksToggle` y el cambio solo existe en el DOM.

### Incertidumbres

- No he observado en vivo la respuesta exacta al GET. Lo esperable es un 404 de Buffalo por ruta inexistente, pero no lo he comprobado; el test de caracterización lo fijará.
- Tampoco lo he reproducido en un navegador sobre esta rama. El diagnóstico sale de leer el código y el historial, que son concluyentes.

### Reproducción mínima

**Manual, con la aplicación arrancada y JavaScript activo:**

1. Abrir `/`.
2. Pulsar completar en una tarea pendiente. Se tacha.
3. Recargar. Vuelve a pendiente.

En las herramientas de red del navegador se ve una petición `GET /tasks/<id>/toggle` con respuesta no-2xx.

**Equivalente HTTP:** un `GET /tasks/<id>/toggle` no cambia `completed`.

## Alcance

**Incluido**

- Corregir en `public/assets/app.js` la petición del toggle para que replique el envío nativo:
  - POST a `form.action`;
  - cuerpo `application/x-www-form-urlencoded` con los campos del formulario (`authenticity_token` y `_method=PUT`);
  - cookies de sesión (`credentials: "same-origin"`).
- Gestionar el fallo: si la respuesta no es OK o la promesa se rechaza, `window.location.reload()`.
- Subir el cache-buster de `app.js` en `templates/application.plush.html` (`?v=2` → `?v=3`).
- Tests en Go de caracterización y de contrato (ver «Tests y validación»).

**Excluido**

- Cambios de rutas, de `TasksToggle` o del middleware CSRF.
- Añadir una ruta GET de toggle, que sería un cambio de estado por GET y quedaría sin CSRF.
- Introducir un runner de tests de JavaScript o de navegador (npm, Playwright…).
- Cambios en `data-confirm`, en el resto de formularios o en el diseño.
- Gestionar dobles clics rápidos. Cada clic envía su propio PUT y el servidor alterna en orden, así que el estado final coincide con el número de clics. No se cambia.

## Archivos relevantes

| Ruta | Responsabilidad |
|---|---|
| `public/assets/app.js` | Intercepta el submit del toggle. **Aquí está el fallo** (línea 34) |
| `templates/application.plush.html` | Carga `app.js?v=2`. Cache-buster a subir |
| `templates/tasks/index.plush.html` | Formulario del toggle con `authenticity_token` y `_method=PUT` (solo lectura) |
| `actions/app.go` | Ruta `PUT /tasks/{task_id}/toggle`, CSRF y method override (solo lectura) |
| `actions/tasks.go` | `TasksToggle` (solo lectura) |
| `actions/tasks_test.go` | Tests nuevos de caracterización y de contrato del asset |

## Implementación paso a paso

1. **`actions/tasks_test.go`: añadir primero los tests** (ver abajo) y comprobar que `Test_AppJS_TogglePersistsWithFormRequest` **falla** con el `app.js` actual.

2. **`public/assets/app.js`: sustituir la línea `fetch(form.action);`** por:

   ```js
   fetch(form.action, {
     method: "POST",
     body: new URLSearchParams(new FormData(form)),
     credentials: "same-origin",
   })
     .then(function (response) {
       if (!response.ok) {
         window.location.reload();
       }
     })
     .catch(function () {
       window.location.reload();
     });
   ```

   El resto del manejador se queda igual: `preventDefault`, actualización optimista del DOM, etiquetas ARIA y `.task-status`.

   - `new FormData(form)` incluye `authenticity_token` y `_method`. `URLSearchParams` lo serializa como `application/x-www-form-urlencoded`, igual que el envío nativo, así que lo leen `PostFormValue` (CSRF) y `FormValue` (method override).
   - `fetch` sigue el 303 a `/`, y `response.ok` refleja la respuesta final (200).

3. **`templates/application.plush.html`:** cambiar `/assets/app.js?v=2` por `/assets/app.js?v=3`.

4. Ejecutar las validaciones y la comprobación manual.

## Tests y validación

**Precondiciones**

- PostgreSQL accesible según `database.yml`.
- `GO_ENV=test`.
- `soda migrate -e test` (soda `v6.4.1`, como en `.github/workflows/ci.yaml`).

**Tests nuevos en `actions/tasks_test.go`**

1. **`Test_TasksToggle_GetDoesNotToggle`** (caracterización de la causa)
   - Crear una tarea pendiente.
   - Hacer `as.HTML("/tasks/%s/toggle", task.ID).Get()`.
   - Afirmar que `res.Code` no es 200 ni 303.
   - Tras `Reload`, afirmar que `task.Completed` sigue en `false`.
   - Pasa antes y después del arreglo. Documenta que un GET nunca persistirá el cambio.
2. **`Test_AppJS_TogglePersistsWithFormRequest`** (regresión, contrato del asset)
   - Hacer `as.HTML("/assets/app.js").Get()` y comprobar el 200. El asset se sirve embebido desde `public`.
   - Afirmar que el cuerpo contiene `method: "POST"`, `new FormData(form)` y `credentials: "same-origin"`.
   - Afirmar que **no** contiene `fetch(form.action);`.
   - **Debe fallar antes del arreglo.**
   - Limitación: comprueba el texto, no ejecuta el JavaScript. Es la única regresión automatizable sin introducir un runner de JS, que queda excluido.

**Tests existentes que deben seguir pasando:** `Test_TasksToggle_CompletesAndReopens`, `Test_TasksToggle_NotFound`, `Test_TasksToggle_InvalidID` y el resto de la suite.

**Comandos**

```
gofmt -l .
go vet ./...
soda migrate -e test
go test -p 1 ./...
go build ./...
```

**Comprobación manual obligatoria (el CSRF real solo está activo fuera de test)**

1. Arrancar la aplicación en desarrollo o con Docker (`/env:start`).
2. En el navegador, con la pestaña de red abierta:
   - pulsar completar → debe salir `POST /tasks/<id>/toggle` con 303 y después `GET /` con 200;
   - recargar → la tarea sigue completada;
   - reabrir y recargar → sigue pendiente.
3. Sin navegador, se puede replicar la petición con `curl`:
   - `GET /` con un *cookie jar*;
   - extraer el `authenticity_token` del formulario;
   - `POST /tasks/<id>/toggle` con `_method=PUT&authenticity_token=<token>` y la cookie → 303;
   - `GET /` → la tarea aparece con `task-completed`.

## Riesgos y recuperación

- **Test de contrato frágil.**
  - Riesgo: depende del texto de `app.js` y un refactor equivalente (p. ej. `method: 'POST'` con comillas simples) lo rompería sin un fallo real.
  - Mitigación: las aserciones se limitan a tres fragmentos estables. Si se refactoriza, se actualizan.
- **Recargas en bucle no esperadas.** `reload()` solo se ejecuta tras un fallo del toggle y no se repite automáticamente: no hay bucle.
- **Caché del navegador.** Sin subir `?v=3`, se seguiría sirviendo el `app.js` antiguo y el fallo continuaría. El paso 3 lo evita.
- **Estado incoherente con varias pestañas.** Fuera de alcance, igual que hoy: el toggle alterna sobre el estado del servidor, no fija un valor absoluto.
- **Recuperación.** El cambio está acotado a un manejador de JS y a un cache-buster. Revertir el commit restaura el comportamiento anterior. No hay migraciones.
