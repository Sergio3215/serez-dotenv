# serez-dotenv

Cargador de archivos `.env` para [Serez-Code](https://serezcode.org). Lee variables de entorno a
nivel proyecto desde `.env`, `.env.local` y `.env.<modo>`, las fusiona por capas de prioridad y
las resuelve en **solo lectura**: nunca muta el entorno real del proceso.

```serez
import "serez-dotenv"

let process = new Process("dev")          // modo: dev | qa | local
let host = process.env.get("HOST")        // valor resuelto, o null si no existe
```

## Instalación

```powershell
sz install serez-dotenv
```

La librería declara el permiso `Env` por sí misma (solo para el fallback al entorno del sistema),
así que tu app **no** necesita declararlo.

## Jerarquía

De menor a mayor prioridad (la más específica gana; los `.env` se leen del directorio de trabajo):

```
System  <  .env  <  .env.local  <  .env.<modo>   (dev | qa)
```

| Modo    | Capas cargadas                          |
|---------|------------------------------------------|
| `dev`   | `.env` + `.env.local` + `.env.dev`       |
| `qa`    | `.env` + `.env.local` + `.env.qa`        |
| `local` | `.env` + `.env.local`                    |

Si ninguna capa define la variable, cae al entorno del **sistema** como último recurso. Es lo
contrario a `Env` del core: no impone nada al sistema, solo prioriza el `.env` al leer.

## API

| Llamada                | Devuelve | Descripción |
|------------------------|----------|-------------|
| `new Process(mode)`    | Process  | Carga las capas del directorio actual según `mode`. |
| `process.env.get(key)` | string?  | Valor resuelto por capas; cae a System; `null` si no existe (no lanza). |
| `process.env.has(key)` | bool     | `true` si la clave está en alguna capa `.env` (ignora System). |
| `process.mode`         | string   | El modo con el que se construyó. |
| `loadEnv(dir, mode)`   | EnvResolver | Carga las capas desde un directorio arbitrario (útil en tests). |

## Formato `.env`

```sh
# comentario (solo línea completa)
HOST=localhost
QUOTED="con espacios"        # comillas envolventes se quitan
export TOKEN=abc123          # el prefijo 'export' se ignora
EMPTY=                       # valor vacío permitido
```

Una variable por línea (`CLAVE=VALOR`), espacios recortados, la última repetida gana. No hay
comentarios en línea.

## Ejemplo completo

`.env` → `HOST=base-host`, `PORT=3000` · `.env.dev` → `HOST=dev-host`, `DEBUG=true`

```serez
import "serez-dotenv"

let process = new Process("dev")
let host  = process.env.get("HOST")    // "dev-host"  (.env.dev pisa a .env)
let port  = process.env.get("PORT")    // "3000"      (solo en .env)
let user  = process.env.get("USER")    // del sistema, o null si no existe
```

## Documentación

- **[Referencia de serez-dotenv](https://serezcode.org/docs/serez-dotenv)** — jerarquía, API y
  formato en detalle, en el sitio de Serez-Code.

## Tests

```powershell
.\run_tests.ps1    # Windows   (./run_tests.sh en Linux/macOS)
```

## Diseño

Solo lectura (nunca `Env.set`), `.sz` puro sin dependencias, estructura modular
(`index.sz` reexporta `src/parser`, `src/resolver`, `src/process`).
