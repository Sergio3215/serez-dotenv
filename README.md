# serez-dotenv

Cargador de archivos `.env` para **Serez-Code**. Lee variables de entorno a nivel
proyecto desde `.env`, `.env.local` y `.env.<modo>`, las fusiona por capas de
prioridad y las resuelve en **solo lectura**: nunca muta el entorno real del proceso.

```serez
import "serez-dotenv"

let process = new Process("dev")          // modo: dev | qa | local
let host = process.env.get("HOST")        // valor resuelto, o null si no existe
```

## Idea

Un `.env` define variables a nivel **proyecto**. Cuando consultás una variable, la
librería mira primero las capas `.env` (la más específica gana) y, si ninguna la
define, cae al entorno del **sistema** como último recurso. Es lo contrario a `Env`
del core: no impone nada al sistema, solo le da prioridad al `.env` al momento de leer.

## Jerarquía

De menor a mayor prioridad:

```
System  <  .env  <  .env.local  <  .env.<modo>   (dev | qa)
```

- `.env` — capa base, se carga siempre (si existe).
- `.env.local` — personalización del desarrollador, se carga siempre. **Siempre pierde**
  contra `.env.dev` / `.env.qa` (es a propósito: overrides personales sin volver robusto al `.local`).
- `.env.<modo>` — capa superior, solo en modo `dev` o `qa`.
- En modo `local` no hay capa `dev`/`qa`: queda `.env < .env.local`.

Ejemplo (modo `dev`, todos definen `HOST`): gana `.env.dev`.

```
System    HOST  ✗
.env      HOST  ✗
.env.local HOST ✗
.env.dev  HOST  ✓   ← gana
```

## Modos

La clase de entrada recibe el **modo** al construirse:

| Modo    | Capas cargadas                          |
|---------|------------------------------------------|
| `dev`   | `.env` + `.env.local` + `.env.dev`       |
| `qa`    | `.env` + `.env.local` + `.env.qa`        |
| `local` | `.env` + `.env.local`                    |

Los `.env` se leen del **directorio de trabajo actual** (la raíz del proyecto).

## API

| Llamada                     | Devuelve | Descripción |
|-----------------------------|----------|-------------|
| `new Process(mode)`         | Process  | Carga las capas del directorio actual según `mode`. |
| `process.env.get(key)`      | string?  | Valor resuelto por capas; cae a System; `null` si no existe en ningún lado. |
| `process.env.has(key)`      | bool     | `true` si la clave está definida en alguna capa `.env` (ignora System). |
| `process.mode`              | string   | El modo con el que se construyó. |
| `loadEnv(dir, mode)`        | EnvResolver | Carga las capas desde un directorio arbitrario. Útil para tests o para leer `.env` de otra carpeta. |

`process.env.get(...)` devuelve `null` (no lanza) cuando la variable no existe en
ninguna capa ni en el sistema.

## Formato de archivo `.env`

```sh
# Los comentarios empiezan con '#' (línea completa)
HOST=localhost
PORT=8080
DB_URL=postgres://user:pass@host/db
QUOTED="con espacios"        # las comillas envolventes se quitan
SINGLE='también simples'
export TOKEN=abc123          # el prefijo 'export' se ignora
EMPTY=                       # valor vacío permitido
```

Reglas:

- Una variable por línea: `CLAVE=VALOR`.
- Líneas vacías y las que empiezan con `#` se ignoran.
- Se recortan los espacios alrededor de la clave y del valor.
- Comillas `"..."` o `'...'` envolventes se eliminan.
- Prefijo opcional `export ` se descarta.
- Si la clave se repite dentro del mismo archivo, gana la última.
- El `#` solo es comentario al **inicio de línea**; no hay comentarios en línea.

## Permisos

La librería declara el permiso `Env` por sí misma (lo usa solo para el fallback al
entorno del sistema), así que el consumidor **no** necesita declararlo. La lectura de
los archivos `.env` usa `File`, que no requiere permiso.

## Ejemplo completo

`.env`
```sh
HOST=base-host
PORT=3000
DB_URL=postgres://localhost/app
```

`.env.dev`
```sh
HOST=dev-host
DEBUG=true
```

`app.sz`
```serez
import "serez-dotenv"

let process = new Process("dev")

let host  = process.env.get("HOST")    // "dev-host"   (.env.dev pisa a .env)
let port  = process.env.get("PORT")    // "3000"       (solo en .env)
let debug = process.env.get("DEBUG")   // "true"        (solo en .env.dev)
let user  = process.env.get("USER")    // del sistema, o null si no existe
```

## Tests

```powershell
.\run_tests.ps1            # Windows
./run_tests.sh             # Linux/macOS
```

Busca `sz` en `../Serez-code/target/release|debug`. Se le puede pasar la ruta:
`.\run_tests.ps1 -sz C:\ruta\a\sz.exe`.

## Notas de diseño

- **Solo lectura**: nunca hace `Env.set`; no contamina el entorno del proceso.
- **Sin dependencias**: `.sz` puro sobre el core (namespaces `File` y `Env`).
- Estructura modular en `src/` (`index.sz` reexporta `src/parser`, `src/resolver`,
  `src/process`). Los imports son relativos al directorio del archivo: `index.sz`
  (en la raíz) importa `"src/parser"`, pero `src/process.sz` importa a sus hermanos
  como `"parser"`/`"resolver"` (no `"src/parser"`).
