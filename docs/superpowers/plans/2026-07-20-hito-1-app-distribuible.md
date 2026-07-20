# JFM PDF Editor — Hito 1: Aplicación distribuible

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Producir un instalador de Windows firmado, auto-actualizable, que ejecuta BentoPDF como aplicación de escritorio con marca JFM, asociación de archivos `.pdf` y guardado in situ.

**Architecture:** Wrapper Electron sobre BentoPDF incorporado como submódulo git con tag fijo. El renderer carga desde un servidor HTTP en loopback que emite cabeceras COOP/COEP (requeridas por `SharedArrayBuffer`, imposible bajo `file://`). El proceso main expone una API IPC estrecha vía `contextBridge` para lectura y escritura de archivos. Distribución por instalador NSIS firmado con Azure Trusted Signing y actualización vía GitHub Releases.

**Tech Stack:** Electron, TypeScript, electron-builder, electron-updater, Vitest (unitarias), Playwright (E2E sobre Electron), GitHub Actions.

**Spec:** [`docs/superpowers/specs/2026-07-20-jfm-pdf-editor-design.md`](../specs/2026-07-20-jfm-pdf-editor-design.md)

## Global Constraints

- **Node.js:** >= 20.
- **Submódulo BentoPDF:** `vendor/bentopdf`, anclado al tag `v2.8.6`. Nunca se modifica su código.
- **Build de BentoPDF:** usar `npm run build:docker` (equivale a `vite build`). **No** usar `npm run build` — su cadena usa sintaxis POSIX (`NODE_OPTIONS='...' node ...`) que falla en Windows. **No** usar `build:production` — activa `VITE_USE_CDN=true`, que carga assets remotos, rompe el funcionamiento offline e incumple COEP.
- **Cabeceras obligatorias en toda respuesta del servidor:** `Cross-Origin-Opener-Policy: same-origin`, `Cross-Origin-Embedder-Policy: require-corp`, `Cross-Origin-Resource-Policy: same-origin`.
- **Seguridad del renderer:** `contextIsolation: true`, `nodeIntegration: false`, `sandbox: true`. Sin excepciones.
- **Servidor:** escucha exclusivamente en `127.0.0.1`, puerto efímero (`0`). Nunca `0.0.0.0`.
- **Escritura de archivos:** el proceso main solo escribe en rutas previamente abiertas por el usuario o elegidas en un diálogo nativo. El renderer nunca decide una ruta arbitraria.
- **Sin telemetría.** Ninguna petición de red salvo la comprobación de actualizaciones a GitHub.
- **Paleta JFM:** naranja `#F6921E`, tinta `#0F0F12`. Tipografía `Plus Jakarta Sans`. Easing `cubic-bezier(0.23,1,0.32,1)`.
- **Nombre de producto:** `JFM PDF Editor`. `appId`: `com.jfmtechnology.pdfeditor`.

---

## Estructura de archivos

| Archivo | Responsabilidad |
|---|---|
| `electron/server.ts` | Servidor HTTP estático en loopback con cabeceras COOP/COEP |
| `electron/files.ts` | Parseo de argv, registro de archivos abiertos, lectura/escritura |
| `electron/preload.ts` | Puente `contextBridge` — única superficie expuesta al renderer |
| `electron/main.ts` | Ciclo de vida, ventana, instancia única, registro de handlers IPC |
| `electron/updater.ts` | Inicialización de `electron-updater` |
| `branding/jfm-brand.css` | Override de paleta y tipografía sobre la UI de BentoPDF |
| `scripts/build-web.mjs` | Construye BentoPDF e inyecta el CSS de marca en el `dist/` |
| `build/electron-builder.yml` | Empaquetado, NSIS, asociación `.pdf`, firma, publicación |
| `.github/workflows/release.yml` | CI: build, firma y publicación por tag |

---

## Task 1: Scaffolding y build de BentoPDF

**Files:**
- Create: `package.json`, `tsconfig.json`, `vitest.config.ts`, `scripts/build-web.mjs`
- Create: `vendor/bentopdf` (submódulo)
- Test: `tests/build-web.test.ts`

**Interfaces:**
- Consumes: nada.
- Produces: `dist-web/` — directorio con el build de BentoPDF, consumido por todas las tareas siguientes.

- [ ] **Step 1: Inicializar el paquete y añadir el submódulo**

```bash
npm init -y
npm pkg set name="jfm-pdf-editor" version="0.1.0" private=true type="module"
npm pkg set description="Editor de PDF de escritorio para Windows"
npm pkg delete main

git submodule add -b main https://github.com/alam00000/bentopdf.git vendor/bentopdf
cd vendor/bentopdf && git checkout v2.8.6 && cd ../..
git add .gitmodules vendor/bentopdf
```

- [ ] **Step 2: Instalar dependencias**

```bash
npm install --save-dev electron@latest electron-builder typescript tsx vitest @types/node
npm install electron-updater
```

- [ ] **Step 3: Crear `tsconfig.json`**

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "resolveJsonModule": true,
    "outDir": "dist-electron",
    "types": ["node"]
  },
  "include": ["electron/**/*.ts", "tests/**/*.ts", "scripts/**/*.mjs"],
  "exclude": ["vendor"]
}
```

- [ ] **Step 4: Crear `vitest.config.ts`**

```ts
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    include: ['tests/**/*.test.ts'],
    environment: 'node',
  },
});
```

- [ ] **Step 5: Escribir el test que falla**

Crear `tests/build-web.test.ts`:

```ts
import { describe, it, expect } from 'vitest';
import { existsSync, readFileSync } from 'node:fs';
import { join } from 'node:path';

const DIST = join(process.cwd(), 'dist-web');

describe('build-web', () => {
  it('produce un index.html', () => {
    expect(existsSync(join(DIST, 'index.html'))).toBe(true);
  });

  it('inyecta la hoja de marca JFM en index.html', () => {
    const html = readFileSync(join(DIST, 'index.html'), 'utf8');
    expect(html).toContain('/jfm-brand.css');
  });

  it('copia la hoja de marca al dist', () => {
    expect(existsSync(join(DIST, 'jfm-brand.css'))).toBe(true);
  });

  it('no referencia CDNs externos', () => {
    const html = readFileSync(join(DIST, 'index.html'), 'utf8');
    expect(html).not.toMatch(/https?:\/\/cdn\./);
  });
});
```

- [ ] **Step 6: Ejecutar el test y verificar que falla**

Run: `npx vitest run tests/build-web.test.ts`
Expected: FAIL — `dist-web/index.html` no existe.

- [ ] **Step 7: Crear el placeholder de marca**

Crear `branding/jfm-brand.css` con contenido mínimo (la Task 6 lo completa):

```css
/* JFM PDF Editor — capa de marca sobre BentoPDF */
:root {
  --jfm-naranja: #F6921E;
  --jfm-tinta: #0F0F12;
}
```

- [ ] **Step 8: Escribir `scripts/build-web.mjs`**

```js
import { execSync } from 'node:child_process';
import { cpSync, readdirSync, readFileSync, writeFileSync, rmSync, existsSync } from 'node:fs';
import { join, dirname } from 'node:path';
import { fileURLToPath } from 'node:url';

const ROOT = join(dirname(fileURLToPath(import.meta.url)), '..');
const VENDOR = join(ROOT, 'vendor', 'bentopdf');
const OUT = join(ROOT, 'dist-web');

// build:docker == `vite build`. No usar `build` (sintaxis POSIX, falla en Windows)
// ni `build:production` (VITE_USE_CDN=true carga assets remotos).
console.log('Instalando dependencias de BentoPDF...');
execSync('npm ci', { cwd: VENDOR, stdio: 'inherit' });

console.log('Construyendo BentoPDF...');
execSync('npm run build:docker', { cwd: VENDOR, stdio: 'inherit' });

if (existsSync(OUT)) rmSync(OUT, { recursive: true, force: true });
cpSync(join(VENDOR, 'dist'), OUT, { recursive: true });

cpSync(join(ROOT, 'branding', 'jfm-brand.css'), join(OUT, 'jfm-brand.css'));

// Inyecta la hoja de marca al final del <head> de cada HTML.
const LINK = '<link rel="stylesheet" href="/jfm-brand.css">';
let injected = 0;
for (const file of readdirSync(OUT).filter((f) => f.endsWith('.html'))) {
  const path = join(OUT, file);
  const html = readFileSync(path, 'utf8');
  if (html.includes('/jfm-brand.css')) continue;
  if (!html.includes('</head>')) {
    throw new Error(`${file} no tiene </head>; no se puede inyectar la marca`);
  }
  writeFileSync(path, html.replace('</head>', `  ${LINK}\n</head>`));
  injected++;
}
console.log(`Marca JFM inyectada en ${injected} archivo(s) HTML.`);
```

- [ ] **Step 9: Registrar el script y ejecutarlo**

```bash
npm pkg set scripts.build:web="node scripts/build-web.mjs"
npm pkg set scripts.test="vitest run"
npm run build:web
```

Expected: termina sin error e imprime `Marca JFM inyectada en N archivo(s) HTML.` con N >= 1.

- [ ] **Step 10: Ejecutar el test y verificar que pasa**

Run: `npx vitest run tests/build-web.test.ts`
Expected: PASS — los 4 tests en verde.

- [ ] **Step 11: Ignorar artefactos y commitear**

```bash
printf 'dist-web/\ndist-electron/\n' >> .gitignore
git add -A
git commit -m "feat: scaffolding y build de BentoPDF con inyección de marca"
```

---

## Task 2: Servidor estático con COOP/COEP

**Files:**
- Create: `electron/server.ts`
- Test: `tests/server.test.ts`

**Interfaces:**
- Consumes: `dist-web/` (Task 1).
- Produces:
  ```ts
  export interface StaticServer {
    url: string;      // p.ej. "http://127.0.0.1:51234"
    port: number;
    close(): Promise<void>;
  }
  export function startStaticServer(root: string): Promise<StaticServer>;
  ```

- [ ] **Step 1: Escribir el test que falla**

Crear `tests/server.test.ts`:

```ts
import { describe, it, expect, beforeAll, afterAll } from 'vitest';
import { mkdtempSync, writeFileSync, mkdirSync } from 'node:fs';
import { tmpdir } from 'node:os';
import { join } from 'node:path';
import { startStaticServer, type StaticServer } from '../electron/server.js';

let server: StaticServer;
let root: string;

beforeAll(async () => {
  root = mkdtempSync(join(tmpdir(), 'jfm-srv-'));
  writeFileSync(join(root, 'index.html'), '<html><head></head><body>hola</body></html>');
  writeFileSync(join(root, 'app.wasm'), Buffer.from([0x00, 0x61, 0x73, 0x6d]));
  mkdirSync(join(root, 'sub'));
  writeFileSync(join(root, 'sub', 'a.js'), 'export const a = 1;');
  server = await startStaticServer(root);
});

afterAll(async () => { await server.close(); });

describe('startStaticServer', () => {
  it('escucha en loopback', () => {
    expect(server.url).toMatch(/^http:\/\/127\.0\.0\.1:\d+$/);
  });

  it('emite las cabeceras de aislamiento de origen cruzado', async () => {
    const res = await fetch(`${server.url}/index.html`);
    expect(res.headers.get('cross-origin-opener-policy')).toBe('same-origin');
    expect(res.headers.get('cross-origin-embedder-policy')).toBe('require-corp');
    expect(res.headers.get('cross-origin-resource-policy')).toBe('same-origin');
  });

  it('sirve / como index.html', async () => {
    const res = await fetch(`${server.url}/`);
    expect(await res.text()).toContain('hola');
  });

  it('sirve .wasm con el MIME correcto', async () => {
    const res = await fetch(`${server.url}/app.wasm`);
    expect(res.headers.get('content-type')).toBe('application/wasm');
  });

  it('sirve subdirectorios', async () => {
    const res = await fetch(`${server.url}/sub/a.js`);
    expect(res.status).toBe(200);
    expect(res.headers.get('content-type')).toContain('javascript');
  });

  it('devuelve 404 para archivos inexistentes', async () => {
    const res = await fetch(`${server.url}/no-existe.txt`);
    expect(res.status).toBe(404);
  });

  it('bloquea path traversal', async () => {
    const res = await fetch(`${server.url}/../../etc/passwd`);
    expect(res.status).toBeGreaterThanOrEqual(400);
  });
});
```

- [ ] **Step 2: Ejecutar el test y verificar que falla**

Run: `npx vitest run tests/server.test.ts`
Expected: FAIL — no se puede resolver `../electron/server.js`.

- [ ] **Step 3: Implementar `electron/server.ts`**

```ts
import { createServer, type Server } from 'node:http';
import { createReadStream } from 'node:fs';
import { stat } from 'node:fs/promises';
import { join, normalize, extname, resolve, sep } from 'node:path';

export interface StaticServer {
  url: string;
  port: number;
  close(): Promise<void>;
}

const MIME: Record<string, string> = {
  '.html': 'text/html; charset=utf-8',
  '.js': 'text/javascript; charset=utf-8',
  '.mjs': 'text/javascript; charset=utf-8',
  '.css': 'text/css; charset=utf-8',
  '.json': 'application/json; charset=utf-8',
  '.wasm': 'application/wasm',
  '.png': 'image/png',
  '.jpg': 'image/jpeg',
  '.jpeg': 'image/jpeg',
  '.gif': 'image/gif',
  '.svg': 'image/svg+xml',
  '.webp': 'image/webp',
  '.ico': 'image/x-icon',
  '.woff': 'font/woff',
  '.woff2': 'font/woff2',
  '.ttf': 'font/ttf',
  '.icc': 'application/vnd.iccprofile',
  '.traineddata': 'application/octet-stream',
  '.pdf': 'application/pdf',
  '.txt': 'text/plain; charset=utf-8',
  '.map': 'application/json; charset=utf-8',
};

// Requeridas por SharedArrayBuffer, del que dependen LibreOffice WASM y wasm-vips.
// Portadas desde vendor/bentopdf/nginx.conf.
function applyIsolationHeaders(res: import('node:http').ServerResponse): void {
  res.setHeader('Cross-Origin-Opener-Policy', 'same-origin');
  res.setHeader('Cross-Origin-Embedder-Policy', 'require-corp');
  res.setHeader('Cross-Origin-Resource-Policy', 'same-origin');
}

export function startStaticServer(root: string): Promise<StaticServer> {
  const rootResolved = resolve(root);

  const server: Server = createServer(async (req, res) => {
    applyIsolationHeaders(res);

    let pathname: string;
    try {
      pathname = decodeURIComponent(new URL(req.url ?? '/', 'http://127.0.0.1').pathname);
    } catch {
      res.statusCode = 400;
      res.end('Bad Request');
      return;
    }

    if (pathname.endsWith('/')) pathname += 'index.html';

    const target = resolve(join(rootResolved, normalize(pathname)));
    if (target !== rootResolved && !target.startsWith(rootResolved + sep)) {
      res.statusCode = 403;
      res.end('Forbidden');
      return;
    }

    try {
      const info = await stat(target);
      if (!info.isFile()) {
        res.statusCode = 404;
        res.end('Not Found');
        return;
      }
      res.setHeader('Content-Type', MIME[extname(target).toLowerCase()] ?? 'application/octet-stream');
      res.setHeader('Content-Length', info.size);
      res.setHeader('Cache-Control', 'no-cache');
      createReadStream(target).pipe(res);
    } catch {
      res.statusCode = 404;
      res.end('Not Found');
    }
  });

  return new Promise((resolvePromise, reject) => {
    server.on('error', reject);
    server.listen(0, '127.0.0.1', () => {
      const addr = server.address();
      if (addr === null || typeof addr === 'string') {
        reject(new Error('El servidor no devolvió una dirección TCP'));
        return;
      }
      resolvePromise({
        url: `http://127.0.0.1:${addr.port}`,
        port: addr.port,
        close: () => new Promise<void>((done) => server.close(() => done())),
      });
    });
  });
}
```

- [ ] **Step 4: Ejecutar el test y verificar que pasa**

Run: `npx vitest run tests/server.test.ts`
Expected: PASS — los 7 tests en verde.

- [ ] **Step 5: Commit**

```bash
git add electron/server.ts tests/server.test.ts
git commit -m "feat: servidor estático en loopback con cabeceras COOP/COEP"
```

---

## Task 3: Lógica de archivos

**Files:**
- Create: `electron/files.ts`
- Test: `tests/files.test.ts`

**Interfaces:**
- Consumes: nada.
- Produces:
  ```ts
  export function parsePdfPathFromArgv(argv: string[]): string | null;
  export class OpenFileRegistry {
    register(filePath: string): void;
    isRegistered(filePath: string): boolean;
  }
  ```

`OpenFileRegistry` es la barrera de seguridad: el proceso main solo sobrescribe archivos que el usuario abrió explícitamente o eligió en un diálogo. Sin ella, un renderer comprometido podría escribir en cualquier ruta.

- [ ] **Step 1: Escribir el test que falla**

Crear `tests/files.test.ts`:

```ts
import { describe, it, expect } from 'vitest';
import { parsePdfPathFromArgv, OpenFileRegistry } from '../electron/files.js';

describe('parsePdfPathFromArgv', () => {
  it('encuentra un .pdf entre los argumentos', () => {
    expect(parsePdfPathFromArgv(['app.exe', 'C:\\docs\\a.pdf'])).toBe('C:\\docs\\a.pdf');
  });

  it('ignora mayúsculas en la extensión', () => {
    expect(parsePdfPathFromArgv(['app.exe', 'C:\\docs\\A.PDF'])).toBe('C:\\docs\\A.PDF');
  });

  it('devuelve null si no hay ningún .pdf', () => {
    expect(parsePdfPathFromArgv(['app.exe', '--inspect'])).toBeNull();
  });

  it('ignora banderas que empiezan por guion', () => {
    expect(parsePdfPathFromArgv(['app.exe', '--flag=x.pdf', 'C:\\real.pdf'])).toBe('C:\\real.pdf');
  });

  it('devuelve null con argv vacío', () => {
    expect(parsePdfPathFromArgv([])).toBeNull();
  });
});

describe('OpenFileRegistry', () => {
  it('reconoce una ruta registrada', () => {
    const r = new OpenFileRegistry();
    r.register('C:\\docs\\a.pdf');
    expect(r.isRegistered('C:\\docs\\a.pdf')).toBe(true);
  });

  it('rechaza una ruta no registrada', () => {
    const r = new OpenFileRegistry();
    expect(r.isRegistered('C:\\otro\\b.pdf')).toBe(false);
  });

  it('compara sin distinguir mayúsculas (Windows)', () => {
    const r = new OpenFileRegistry();
    r.register('C:\\Docs\\A.pdf');
    expect(r.isRegistered('c:\\docs\\a.pdf')).toBe(true);
  });
});
```

- [ ] **Step 2: Ejecutar el test y verificar que falla**

Run: `npx vitest run tests/files.test.ts`
Expected: FAIL — no se puede resolver `../electron/files.js`.

- [ ] **Step 3: Implementar `electron/files.ts`**

```ts
import { resolve } from 'node:path';

export function parsePdfPathFromArgv(argv: string[]): string | null {
  for (const arg of argv.slice(1)) {
    if (arg.startsWith('-')) continue;
    if (arg.toLowerCase().endsWith('.pdf')) return arg;
  }
  return null;
}

/**
 * Rutas que el usuario abrió o eligió explícitamente.
 * El proceso main solo escribe en rutas presentes aquí.
 */
export class OpenFileRegistry {
  private readonly paths = new Set<string>();

  private key(filePath: string): string {
    return resolve(filePath).toLowerCase();
  }

  register(filePath: string): void {
    this.paths.add(this.key(filePath));
  }

  isRegistered(filePath: string): boolean {
    return this.paths.has(this.key(filePath));
  }
}
```

- [ ] **Step 4: Ejecutar el test y verificar que pasa**

Run: `npx vitest run tests/files.test.ts`
Expected: PASS — los 8 tests en verde.

- [ ] **Step 5: Commit**

```bash
git add electron/files.ts tests/files.test.ts
git commit -m "feat: parseo de argv y registro de archivos abiertos"
```

---

## Task 4: Preload y contrato IPC

**Files:**
- Create: `electron/preload.ts`
- Create: `electron/ipc-contract.ts`

**Interfaces:**
- Consumes: nada.
- Produces: canales IPC y el objeto `window.jfmPdf`:
  ```ts
  export const IPC = {
    startupFile: 'jfm:startup-file',
    readFile: 'jfm:read-file',
    saveFile: 'jfm:save-file',
    saveFileAs: 'jfm:save-file-as',
    openFileEvent: 'jfm:open-file-event',
  } as const;

  window.jfmPdf: {
    getStartupFile(): Promise<string | null>;
    readFile(filePath: string): Promise<Uint8Array>;
    saveFile(filePath: string, data: Uint8Array): Promise<void>;
    saveFileAs(defaultName: string, data: Uint8Array): Promise<string | null>;
    onOpenFile(callback: (filePath: string) => void): void;
  }
  ```

- [ ] **Step 1: Crear `electron/ipc-contract.ts`**

```ts
export const IPC = {
  startupFile: 'jfm:startup-file',
  readFile: 'jfm:read-file',
  saveFile: 'jfm:save-file',
  saveFileAs: 'jfm:save-file-as',
  openFileEvent: 'jfm:open-file-event',
} as const;

export interface JfmPdfApi {
  getStartupFile(): Promise<string | null>;
  readFile(filePath: string): Promise<Uint8Array>;
  saveFile(filePath: string, data: Uint8Array): Promise<void>;
  saveFileAs(defaultName: string, data: Uint8Array): Promise<string | null>;
  onOpenFile(callback: (filePath: string) => void): void;
}
```

- [ ] **Step 2: Crear `electron/preload.ts`**

```ts
import { contextBridge, ipcRenderer } from 'electron';
import { IPC, type JfmPdfApi } from './ipc-contract.js';

const api: JfmPdfApi = {
  getStartupFile: () => ipcRenderer.invoke(IPC.startupFile),
  readFile: (filePath) => ipcRenderer.invoke(IPC.readFile, filePath),
  saveFile: (filePath, data) => ipcRenderer.invoke(IPC.saveFile, filePath, data),
  saveFileAs: (defaultName, data) => ipcRenderer.invoke(IPC.saveFileAs, defaultName, data),
  onOpenFile: (callback) => {
    ipcRenderer.on(IPC.openFileEvent, (_event, filePath: string) => callback(filePath));
  },
};

contextBridge.exposeInMainWorld('jfmPdf', api);
```

- [ ] **Step 3: Verificar que compila**

Run: `npx tsc --noEmit`
Expected: sin errores.

- [ ] **Step 4: Commit**

```bash
git add electron/preload.ts electron/ipc-contract.ts
git commit -m "feat: contrato IPC y puente contextBridge"
```

---

## Task 5: Proceso main y ventana

**Files:**
- Create: `electron/main.ts`
- Modify: `package.json` (campo `main`, scripts)

**Interfaces:**
- Consumes: `startStaticServer` (Task 2), `parsePdfPathFromArgv` / `OpenFileRegistry` (Task 3), `IPC` (Task 4).
- Produces: aplicación Electron ejecutable con `npm start`.

- [ ] **Step 1: Implementar `electron/main.ts`**

```ts
import { app, BrowserWindow, ipcMain, dialog } from 'electron';
import { readFile, writeFile } from 'node:fs/promises';
import { join, dirname, basename } from 'node:path';
import { fileURLToPath } from 'node:url';
import { startStaticServer, type StaticServer } from './server.js';
import { parsePdfPathFromArgv, OpenFileRegistry } from './files.js';
import { IPC } from './ipc-contract.js';

const __dirname = dirname(fileURLToPath(import.meta.url));

let server: StaticServer | null = null;
let mainWindow: BrowserWindow | null = null;
let startupFile: string | null = null;
const registry = new OpenFileRegistry();

function webRoot(): string {
  return app.isPackaged
    ? join(process.resourcesPath, 'web')
    : join(__dirname, '..', 'dist-web');
}

async function createWindow(): Promise<void> {
  mainWindow = new BrowserWindow({
    width: 1400,
    height: 900,
    minWidth: 900,
    minHeight: 600,
    backgroundColor: '#0F0F12',
    show: false,
    title: 'JFM PDF Editor',
    webPreferences: {
      contextIsolation: true,
      nodeIntegration: false,
      sandbox: true,
      preload: join(__dirname, 'preload.cjs'),
    },
  });

  mainWindow.once('ready-to-show', () => mainWindow?.show());
  await mainWindow.loadURL(server!.url);
}

function registerIpcHandlers(): void {
  ipcMain.handle(IPC.startupFile, () => startupFile);

  ipcMain.handle(IPC.readFile, async (_event, filePath: string) => {
    if (!registry.isRegistered(filePath)) {
      throw new Error('Ruta no autorizada');
    }
    return new Uint8Array(await readFile(filePath));
  });

  // Guardado in situ: sobrescribe el archivo original, sin pasar por Descargas.
  ipcMain.handle(IPC.saveFile, async (_event, filePath: string, data: Uint8Array) => {
    if (!registry.isRegistered(filePath)) {
      throw new Error('Ruta no autorizada');
    }
    await writeFile(filePath, Buffer.from(data));
  });

  ipcMain.handle(IPC.saveFileAs, async (_event, defaultName: string, data: Uint8Array) => {
    const result = await dialog.showSaveDialog(mainWindow!, {
      defaultPath: defaultName,
      filters: [{ name: 'PDF', extensions: ['pdf'] }],
    });
    if (result.canceled || result.filePath === undefined) return null;
    registry.register(result.filePath);
    await writeFile(result.filePath, Buffer.from(data));
    return result.filePath;
  });
}

function adoptFile(filePath: string): void {
  registry.register(filePath);
  startupFile = filePath;
  mainWindow?.webContents.send(IPC.openFileEvent, filePath);
}

// Instancia única: un segundo .pdf se entrega a la ventana existente.
if (!app.requestSingleInstanceLock()) {
  app.quit();
} else {
  app.on('second-instance', (_event, argv) => {
    const filePath = parsePdfPathFromArgv(argv);
    if (filePath !== null) adoptFile(filePath);
    if (mainWindow !== null) {
      if (mainWindow.isMinimized()) mainWindow.restore();
      mainWindow.focus();
    }
  });

  app.whenReady().then(async () => {
    const initial = parsePdfPathFromArgv(process.argv);
    if (initial !== null) {
      registry.register(initial);
      startupFile = initial;
    }

    registerIpcHandlers();

    try {
      server = await startStaticServer(webRoot());
    } catch (error) {
      dialog.showErrorBox(
        'No se pudo iniciar JFM PDF Editor',
        'No fue posible abrir el servidor local de la aplicación.\n\n' +
          'Causa habitual: un antivirus o firewall bloqueando conexiones locales.\n\n' +
          `Detalle: ${String(error)}`,
      );
      app.quit();
      return;
    }

    await createWindow();
  });

  app.on('window-all-closed', async () => {
    await server?.close();
    app.quit();
  });
}
```

- [ ] **Step 2: Empaquetar el preload como CommonJS**

Con `sandbox: true` el preload debe ser CommonJS auténtico. Compilarlo con `tsc` (que emite ESM según nuestro `tsconfig.json`) y renombrarlo a `.cjs` produciría un archivo con sintaxis `import` que Node rechazaría. Hay que bundlearlo con esbuild, que además inlinea `ipc-contract.ts` — necesario porque un preload en sandbox no puede resolver módulos propios.

```bash
npm install --save-dev esbuild

npm pkg set main="dist-electron/main.js"
npm pkg set scripts.build:preload="esbuild electron/preload.ts --bundle --platform=node --format=cjs --external:electron --outfile=dist-electron/preload.cjs"
npm pkg set scripts.build:electron="tsc && npm run build:preload"
npm pkg set scripts.start="npm run build:web && npm run build:electron && electron ."
```

Verificar que el resultado es CJS:

```bash
npm run build:electron
node -e "const s=require('node:fs').readFileSync('dist-electron/preload.cjs','utf8');if(/^\s*import\s/m.test(s))throw new Error('el preload salió como ESM');console.log('preload OK: CommonJS');"
```

Expected: `preload OK: CommonJS`.

- [ ] **Step 3: Ejecutar la aplicación**

Run: `npm start`
Expected: se abre una ventana con la interfaz de BentoPDF sobre fondo oscuro.

- [ ] **Step 4: Verificar manualmente el aislamiento de origen cruzado**

Abrir DevTools con `Ctrl+Shift+I`, y en la consola:

```js
crossOriginIsolated
```

Expected: `true`. Si devuelve `false`, LibreOffice WASM y vips no funcionarán — revisar las cabeceras de la Task 2 antes de continuar.

- [ ] **Step 5: Añadir el menú con el aviso AGPL**

La AGPL exige que el aviso de licencia y el enlace al código fuente sean visibles en la aplicación, no solo en el repositorio.

Crear `electron/menu.ts`:

```ts
import { app, Menu, dialog, shell, type BrowserWindow } from 'electron';

const REPO = 'https://github.com/stingeer007/jfm-pdf-editor';

export function buildAppMenu(window: BrowserWindow): void {
  const menu = Menu.buildFromTemplate([
    {
      label: 'Archivo',
      submenu: [{ role: 'quit', label: 'Salir' }],
    },
    {
      label: 'Ver',
      submenu: [
        { role: 'reload', label: 'Recargar' },
        { role: 'toggleDevTools', label: 'Herramientas de desarrollo' },
        { type: 'separator' },
        { role: 'resetZoom', label: 'Zoom normal' },
        { role: 'zoomIn', label: 'Acercar' },
        { role: 'zoomOut', label: 'Alejar' },
        { type: 'separator' },
        { role: 'togglefullscreen', label: 'Pantalla completa' },
      ],
    },
    {
      label: 'Ayuda',
      submenu: [
        {
          label: 'Código fuente',
          click: () => void shell.openExternal(REPO),
        },
        {
          label: 'Acerca de JFM PDF Editor',
          click: async () => {
            await dialog.showMessageBox(window, {
              type: 'info',
              title: 'Acerca de JFM PDF Editor',
              message: `JFM PDF Editor ${app.getVersion()}`,
              detail:
                'Desarrollado por JFM Technology.\n\n' +
                'Basado en BentoPDF (github.com/alam00000/bentopdf).\n\n' +
                'Este programa es software libre bajo la licencia GNU Affero General ' +
                'Public License v3.0. Se distribuye SIN NINGUNA GARANTÍA.\n\n' +
                `Código fuente: ${REPO}\n\n` +
                'Todo el procesamiento ocurre en este equipo. Ningún documento se envía a servidores.',
              buttons: ['Cerrar', 'Ver código fuente'],
              defaultId: 0,
              cancelId: 0,
            }).then((result) => {
              if (result.response === 1) void shell.openExternal(REPO);
            });
          },
        },
      ],
    },
  ]);
  Menu.setApplicationMenu(menu);
}
```

Importarlo en `electron/main.ts` junto a los demás imports:

```ts
import { buildAppMenu } from './menu.js';
```

E invocarlo dentro de `createWindow()`, justo antes de `await mainWindow.loadURL(...)`:

```ts
  buildAppMenu(mainWindow);
```

- [ ] **Step 6: Verificar el aviso**

Run: `npm start`
Expected: menú `Ayuda → Acerca de JFM PDF Editor` muestra la versión, el aviso AGPL, la atribución a BentoPDF y el enlace al código fuente.

- [ ] **Step 7: Commit**

```bash
git add electron/main.ts electron/menu.ts package.json
git commit -m "feat: proceso main, ventana, instancia única y aviso AGPL"
```

---

## Task 6: Capa de marca JFM

**Files:**
- Modify: `branding/jfm-brand.css`
- Test: `tests/branding.test.ts`

**Interfaces:**
- Consumes: `dist-web/` (Task 1).
- Produces: hoja de estilos de marca inyectada en todos los HTML.

Referencia de los valores originales de BentoPDF, tomados de `vendor/bentopdf/src/css/styles.css`: fondo `#111827`, cards `#1f2937`, bordes `#374151`, acento `#4f46e5`.

- [ ] **Step 1: Escribir el test que falla**

Crear `tests/branding.test.ts`:

```ts
import { describe, it, expect } from 'vitest';
import { readFileSync } from 'node:fs';
import { join } from 'node:path';

const css = readFileSync(join(process.cwd(), 'branding', 'jfm-brand.css'), 'utf8');

describe('jfm-brand.css', () => {
  it('define el naranja JFM', () => {
    expect(css).toContain('#F6921E');
  });

  it('define la tinta JFM', () => {
    expect(css).toContain('#0F0F12');
  });

  it('remapea el acento índigo de BentoPDF', () => {
    expect(css).toContain('--color-indigo-600');
  });

  it('usa Plus Jakarta Sans', () => {
    expect(css).toContain('Plus Jakarta Sans');
  });

  it('no usa naranja como color de texto sobre fondo claro', () => {
    // El naranja JFM da ~2.5:1 sobre claro — por debajo del mínimo WCAG de 4.5:1.
    expect(css).not.toMatch(/color:\s*#F6921E[^;]*;\s*\/\*\s*texto/i);
  });
});
```

- [ ] **Step 2: Ejecutar el test y verificar que falla**

Run: `npx vitest run tests/branding.test.ts`
Expected: FAIL — faltan `--color-indigo-600` y `Plus Jakarta Sans`.

- [ ] **Step 3: Escribir `branding/jfm-brand.css`**

```css
/* JFM PDF Editor — capa de marca sobre BentoPDF.
   No modifica el upstream: solo redefine tokens y sobrescribe los hex
   hardcodeados de vendor/bentopdf/src/css/styles.css. */

@import url('https://fonts.bunny.net/css?family=plus-jakarta-sans:400,500,600,700,800');

:root {
  --jfm-naranja: #F6921E;
  --jfm-naranja-claro: #F89C3A;
  --jfm-naranja-oscuro: #F48123;
  --jfm-tinta: #0F0F12;
  --jfm-superficie: #17171C;
  --jfm-borde: #2A2A32;
  --jfm-ease: cubic-bezier(0.23, 1, 0.32, 1);

  /* Remapea la paleta Tailwind que BentoPDF usa como acento. */
  --color-indigo-400: var(--jfm-naranja-claro);
  --color-indigo-500: var(--jfm-naranja);
  --color-indigo-600: var(--jfm-naranja);
  --color-indigo-700: var(--jfm-naranja-oscuro);
}

body {
  font-family: 'Plus Jakarta Sans', ui-sans-serif, system-ui, sans-serif !important;
  background-color: var(--jfm-tinta) !important;
}

/* Sustituye los hex hardcodeados del upstream. */
.tool-card {
  background-color: var(--jfm-superficie) !important;
  border-color: var(--jfm-borde) !important;
  transition:
    transform 0.2s var(--jfm-ease),
    box-shadow 0.2s var(--jfm-ease),
    border-color 0.2s var(--jfm-ease) !important;
}

.tool-card:hover {
  border-color: var(--jfm-naranja) !important;
}

/* El naranja JFM da ~2.5:1 sobre fondo claro — insuficiente para texto (WCAG 4.5:1).
   Sobre la tinta alcanza ~7:1. Se usa como relleno y borde, no como color de texto
   salvo sobre fondo oscuro. */
a:hover,
.text-indigo-400,
.text-indigo-500 {
  color: var(--jfm-naranja-claro) !important;
}

button:focus-visible,
a:focus-visible {
  outline: 2px solid var(--jfm-naranja) !important;
  outline-offset: 2px;
}
```

- [ ] **Step 4: Eliminar la dependencia de red de la fuente**

El `@import` remoto rompe el funcionamiento offline e incumple COEP. Sustituirlo por la fuente local que ya trae BentoPDF como dependencia:

```bash
npm install @fontsource/plus-jakarta-sans
node -e "const{cpSync,mkdirSync}=require('node:fs');mkdirSync('branding/fonts',{recursive:true});cpSync('node_modules/@fontsource/plus-jakarta-sans/files','branding/fonts',{recursive:true});"
```

Reemplazar la línea `@import url('https://fonts.bunny.net/...')` por:

```css
@font-face {
  font-family: 'Plus Jakarta Sans';
  font-style: normal;
  font-weight: 400 800;
  font-display: swap;
  src: url('/fonts/plus-jakarta-sans-latin-variable-wghtOnly-normal.woff2') format('woff2-variations');
}
```

Añadir la copia de fuentes a `scripts/build-web.mjs`, justo después de la copia del CSS de marca:

```js
cpSync(join(ROOT, 'branding', 'fonts'), join(OUT, 'fonts'), { recursive: true });
```

- [ ] **Step 5: Verificar el nombre real del archivo de fuente**

Run: `ls branding/fonts | grep variable`
Expected: un archivo `.woff2` variable. Ajustar la URL del `@font-face` al nombre exacto que aparezca.

- [ ] **Step 6: Ejecutar el test y verificar que pasa**

Run: `npx vitest run tests/branding.test.ts`
Expected: PASS — los 5 tests en verde.

- [ ] **Step 7: Verificar visualmente**

Run: `npm start`
Expected: interfaz con acentos naranja `#F6921E`, fondo negro `#0F0F12`, tipografía Plus Jakarta Sans. Sin peticiones de red — confirmar en la pestaña Network de DevTools que no hay dominios externos.

- [ ] **Step 8: Commit**

```bash
git add branding/ scripts/build-web.mjs tests/branding.test.ts package.json
git commit -m "feat: capa de marca JFM con fuente empaquetada"
```

---

## Task 7: Empaquetado NSIS y asociación de archivos

**Files:**
- Create: `build/electron-builder.yml`
- Create: `build/icon.ico`
- Modify: `package.json` (scripts)

**Interfaces:**
- Consumes: `dist-web/` (Task 1), `dist-electron/` (Task 5).
- Produces: `release/JFM PDF Editor Setup <version>.exe`.

- [ ] **Step 1: Preparar el icono**

Colocar en `build/icon.ico` un icono multi-resolución (16, 32, 48, 64, 128, 256 px). Debe existir antes de empaquetar; electron-builder falla sin él.

- [ ] **Step 2: Crear `build/electron-builder.yml`**

```yaml
appId: com.jfmtechnology.pdfeditor
productName: JFM PDF Editor
copyright: Copyright © 2026 JFM Technology. AGPL-3.0.

directories:
  output: release
  buildResources: build

files:
  - dist-electron/**/*
  - package.json

extraResources:
  - from: dist-web
    to: web

win:
  target:
    - target: nsis
      arch: [x64]
  icon: build/icon.ico
  fileAssociations:
    - ext: pdf
      name: PDF Document
      description: Documento PDF
      icon: build/icon.ico
      role: Editor

nsis:
  oneClick: false
  allowToChangeInstallationDirectory: true
  perMachine: false
  createDesktopShortcut: true
  createStartMenuShortcut: true
  shortcutName: JFM PDF Editor
  license: LICENSE
  differentialPackage: true

publish:
  provider: github
  owner: stingeer007
  repo: jfm-pdf-editor
```

`perMachine: false` instala por usuario y evita el prompt de UAC. `differentialPackage: true` habilita las actualizaciones delta.

- [ ] **Step 3: Registrar el script de empaquetado**

```bash
npm pkg set scripts.pack="npm run build:web && npm run build:electron && electron-builder --config build/electron-builder.yml --publish never"
```

- [ ] **Step 4: Empaquetar**

Run: `npm run pack`
Expected: se genera `release/JFM PDF Editor Setup 0.1.0.exe`.

- [ ] **Step 5: Verificar la instalación y la asociación de archivos**

1. Ejecutar el instalador generado.
2. Confirmar que aparece en el menú Inicio como "JFM PDF Editor".
3. Clic derecho sobre cualquier `.pdf` → "Abrir con" → debe listar JFM PDF Editor.
4. Abrir un `.pdf` con la aplicación y confirmar que la ruta llega al renderer:

```js
await window.jfmPdf.getStartupFile()
```

Expected: la ruta absoluta del PDF abierto.

- [ ] **Step 6: Commit**

```bash
printf 'release/\n' >> .gitignore
git add build/ package.json .gitignore
git commit -m "feat: empaquetado NSIS con asociación de archivos .pdf"
```

---

## Task 8: Firma con Azure Trusted Signing

**Files:**
- Modify: `build/electron-builder.yml`

**Interfaces:**
- Consumes: configuración de empaquetado (Task 7).
- Produces: instalador firmado, verificable con `signtool verify`.

**Prerequisito:** cuenta de Azure Trusted Signing aprobada para JFM Technology, con los valores de `endpoint`, `codeSigningAccountName` y `certificateProfileName`. Sin ellos esta tarea no puede completarse — verificar elegibilidad antes de empezar.

- [ ] **Step 1: Definir las variables de entorno de firma**

En la máquina de desarrollo (y como secretos en CI):

```
AZURE_TENANT_ID
AZURE_CLIENT_ID
AZURE_CLIENT_SECRET
AZURE_SIGN_ENDPOINT              # p.ej. https://eus.codesigning.azure.net
AZURE_CODE_SIGNING_ACCOUNT_NAME
AZURE_CERT_PROFILE_NAME
```

- [ ] **Step 2: Añadir la configuración de firma a `build/electron-builder.yml`**

Dentro del bloque `win:`, añadir:

```yaml
  azureSignOptions:
    publisherName: JFM Technology
    endpoint: ${env.AZURE_SIGN_ENDPOINT}
    codeSigningAccountName: ${env.AZURE_CODE_SIGNING_ACCOUNT_NAME}
    certificateProfileName: ${env.AZURE_CERT_PROFILE_NAME}
```

- [ ] **Step 3: Verificar que electron-builder soporta `azureSignOptions`**

Run: `npx electron-builder --version`

Comprobar que la versión instalada documenta `azureSignOptions`. Si no lo soporta, actualizar:

```bash
npm install --save-dev electron-builder@latest
```

Si tras actualizar sigue sin soportarlo, usar el hook alternativo: crear `build/sign.js` que invoque `signtool.exe sign /v /debug /fd SHA256 /tr <endpoint> /td SHA256 /dlib <Azure.CodeSigning.Dlib.dll> /dmdf <metadata.json>` y referenciarlo con `win.sign: build/sign.js`.

- [ ] **Step 4: Empaquetar firmado**

Run: `npm run pack`
Expected: el build reporta la firma de los artefactos sin errores.

- [ ] **Step 5: Verificar la firma**

Run:
```powershell
signtool verify /pa /v "release\JFM PDF Editor Setup 0.1.0.exe"
```
Expected: `Successfully verified`, con el sujeto `JFM Technology`.

- [ ] **Step 6: Verificar el comportamiento de SmartScreen**

Descargar el instalador desde una ubicación externa (no la ruta local) y ejecutarlo.
Expected: sin la pantalla "Windows protegió su PC". Si aparece, la reputación aún no se ha propagado — documentarlo y continuar; no bloquea el hito.

- [ ] **Step 7: Commit**

```bash
git add build/electron-builder.yml
git commit -m "feat: firma de código con Azure Trusted Signing"
```

---

## Task 9: Actualización automática

**Files:**
- Create: `electron/updater.ts`
- Modify: `electron/main.ts`

**Interfaces:**
- Consumes: `BrowserWindow` (Task 5), configuración `publish` (Task 7).
- Produces:
  ```ts
  export function initAutoUpdater(window: BrowserWindow): void;
  ```

- [ ] **Step 1: Implementar `electron/updater.ts`**

```ts
import { BrowserWindow, dialog } from 'electron';
import electronUpdater from 'electron-updater';

const { autoUpdater } = electronUpdater;

const SEIS_HORAS = 6 * 60 * 60 * 1000;

export function initAutoUpdater(window: BrowserWindow): void {
  autoUpdater.autoDownload = true;
  autoUpdater.autoInstallOnAppQuit = true;

  // Un fallo de actualización nunca debe impedir usar la aplicación.
  autoUpdater.on('error', (error) => {
    console.error('[updater]', error);
  });

  autoUpdater.on('update-downloaded', async (info) => {
    const result = await dialog.showMessageBox(window, {
      type: 'info',
      buttons: ['Reiniciar ahora', 'Más tarde'],
      defaultId: 1,
      cancelId: 1,
      title: 'Actualización disponible',
      message: `JFM PDF Editor ${info.version} está listo para instalarse.`,
      detail: 'La actualización se aplicará al reiniciar la aplicación.',
    });
    if (result.response === 0) autoUpdater.quitAndInstall();
  });

  void autoUpdater.checkForUpdates().catch(() => undefined);
  setInterval(() => {
    void autoUpdater.checkForUpdates().catch(() => undefined);
  }, SEIS_HORAS);
}
```

- [ ] **Step 2: Invocarlo desde `electron/main.ts`**

Añadir el import junto a los demás:

```ts
import { initAutoUpdater } from './updater.js';
```

Y dentro de `app.whenReady().then(...)`, tras `await createWindow();`:

```ts
    if (app.isPackaged && mainWindow !== null) {
      initAutoUpdater(mainWindow);
    }
```

La guarda `app.isPackaged` evita que el updater se ejecute en desarrollo, donde no hay firma ni versión publicada.

- [ ] **Step 3: Verificar que compila**

Run: `npx tsc --noEmit`
Expected: sin errores.

- [ ] **Step 4: Verificar que no interfiere en desarrollo**

Run: `npm start`
Expected: la aplicación arranca con normalidad y no aparece ningún diálogo de actualización.

- [ ] **Step 5: Commit**

```bash
git add electron/updater.ts electron/main.ts
git commit -m "feat: actualización automática vía GitHub Releases"
```

---

## Task 10: Prueba E2E del recorrido crítico

**Files:**
- Create: `tests/e2e/app.spec.ts`
- Create: `playwright.config.ts`

**Interfaces:**
- Consumes: la aplicación completa (Tasks 1-9).
- Produces: verificación automatizada de aislamiento de origen cruzado y guardado in situ.

Esta es la prueba que más veces evitará una regresión: si `crossOriginIsolated` deja de ser `true`, LibreOffice WASM y vips dejan de funcionar y media aplicación queda inservible sin que nada más lo delate.

- [ ] **Step 1: Instalar Playwright**

```bash
npm install --save-dev @playwright/test
```

- [ ] **Step 2: Crear `playwright.config.ts`**

```ts
import { defineConfig } from '@playwright/test';

export default defineConfig({
  testDir: './tests/e2e',
  timeout: 120_000,
  fullyParallel: false,
  workers: 1,
  reporter: 'list',
});
```

- [ ] **Step 3: Escribir el test que falla**

Crear `tests/e2e/app.spec.ts`:

```ts
import { test, expect, _electron as electron, type ElectronApplication } from '@playwright/test';
import { mkdtempSync, writeFileSync, readFileSync } from 'node:fs';
import { tmpdir } from 'node:os';
import { join } from 'node:path';

// PDF mínimo válido de una página.
const PDF_MINIMO = Buffer.from(
  '%PDF-1.4\n' +
    '1 0 obj<</Type/Catalog/Pages 2 0 R>>endobj\n' +
    '2 0 obj<</Type/Pages/Kids[3 0 R]/Count 1>>endobj\n' +
    '3 0 obj<</Type/Page/Parent 2 0 R/MediaBox[0 0 612 792]>>endobj\n' +
    'trailer<</Root 1 0 R>>\n%%EOF\n',
  'latin1',
);

let app: ElectronApplication;
let pdfPath: string;

test.beforeAll(async () => {
  const dir = mkdtempSync(join(tmpdir(), 'jfm-e2e-'));
  pdfPath = join(dir, 'prueba.pdf');
  writeFileSync(pdfPath, PDF_MINIMO);

  app = await electron.launch({ args: ['.', pdfPath] });
});

test.afterAll(async () => { await app.close(); });

test('el renderer está aislado a nivel de origen cruzado', async () => {
  const page = await app.firstWindow();
  await page.waitForLoadState('domcontentloaded');
  const isolated = await page.evaluate(() => crossOriginIsolated);
  expect(isolated).toBe(true);
});

test('el archivo del argumento de línea de comandos llega al renderer', async () => {
  const page = await app.firstWindow();
  const startup = await page.evaluate(() => window.jfmPdf.getStartupFile());
  expect(startup).toBe(pdfPath);
});

test('el guardado in situ sobrescribe el archivo original', async () => {
  const page = await app.firstWindow();
  const contenidoNuevo = Buffer.concat([PDF_MINIMO, Buffer.from('\n% modificado\n')]);

  await page.evaluate(
    async ({ path, bytes }) => {
      await window.jfmPdf.saveFile(path, new Uint8Array(bytes));
    },
    { path: pdfPath, bytes: Array.from(contenidoNuevo) },
  );

  expect(readFileSync(pdfPath).toString('latin1')).toContain('% modificado');
});

test('rechaza escrituras en rutas no autorizadas', async () => {
  const page = await app.firstWindow();
  const rechazado = await page.evaluate(async () => {
    try {
      await window.jfmPdf.saveFile('C:\\Windows\\System32\\malicioso.pdf', new Uint8Array([1]));
      return false;
    } catch {
      return true;
    }
  });
  expect(rechazado).toBe(true);
});
```

- [ ] **Step 4: Declarar el tipo global de `window.jfmPdf`**

Crear `tests/e2e/global.d.ts`:

```ts
import type { JfmPdfApi } from '../../electron/ipc-contract.js';

declare global {
  interface Window {
    jfmPdf: JfmPdfApi;
  }
}

export {};
```

- [ ] **Step 5: Ejecutar el test y verificar que falla**

Run: `npx playwright test`
Expected: FAIL — la aplicación no está construida todavía en esta sesión.

- [ ] **Step 6: Construir y volver a ejecutar**

```bash
npm run build:web && npm run build:electron
npx playwright test
```

Expected: PASS — los 4 tests en verde. Si `crossOriginIsolated` es `false`, revisar las cabeceras de la Task 2.

- [ ] **Step 7: Registrar el script**

```bash
npm pkg set scripts.test:e2e="playwright test"
```

- [ ] **Step 8: Commit**

```bash
git add tests/e2e playwright.config.ts package.json
git commit -m "test: E2E de aislamiento de origen cruzado y guardado in situ"
```

---

## Task 11: CI y publicación de releases

**Files:**
- Create: `.github/workflows/ci.yml`
- Create: `.github/workflows/release.yml`

**Interfaces:**
- Consumes: todos los scripts de npm definidos en las tareas anteriores.
- Produces: release publicado en GitHub con el instalador firmado.

- [ ] **Step 1: Crear `.github/workflows/ci.yml`**

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:

jobs:
  test:
    runs-on: windows-latest
    steps:
      - uses: actions/checkout@v4
        with:
          submodules: recursive

      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm

      - run: npm ci

      - run: npm run build:web

      - run: npm run build:electron

      - run: npm test

      - run: npx playwright test
```

- [ ] **Step 2: Crear `.github/workflows/release.yml`**

```yaml
name: Release

on:
  push:
    tags: ['v*']

permissions:
  contents: write

jobs:
  release:
    runs-on: windows-latest
    steps:
      - uses: actions/checkout@v4
        with:
          submodules: recursive

      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm

      - run: npm ci

      - run: npm run build:web

      - run: npm run build:electron

      - run: npm test

      - name: Empaquetar, firmar y publicar
        run: npx electron-builder --config build/electron-builder.yml --publish always
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          AZURE_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
          AZURE_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
          AZURE_CLIENT_SECRET: ${{ secrets.AZURE_CLIENT_SECRET }}
          AZURE_SIGN_ENDPOINT: ${{ secrets.AZURE_SIGN_ENDPOINT }}
          AZURE_CODE_SIGNING_ACCOUNT_NAME: ${{ secrets.AZURE_CODE_SIGNING_ACCOUNT_NAME }}
          AZURE_CERT_PROFILE_NAME: ${{ secrets.AZURE_CERT_PROFILE_NAME }}
```

- [ ] **Step 3: Cargar los secretos en GitHub**

En `Settings → Secrets and variables → Actions` del repositorio, añadir los seis secretos `AZURE_*` de la Task 8. `GITHUB_TOKEN` es automático.

- [ ] **Step 4: Verificar el CI**

```bash
git add .github/
git commit -m "ci: workflows de integración y publicación"
git push
```

Expected: el workflow CI pasa en verde en GitHub Actions.

- [ ] **Step 5: Publicar la primera versión**

```bash
npm version 0.1.0 --no-git-tag-version
git add package.json
git commit -m "chore: versión 0.1.0"
git tag v0.1.0
git push && git push --tags
```

Expected: el workflow Release genera y publica el instalador firmado en GitHub Releases.

- [ ] **Step 6: Verificar el release**

1. Abrir la página de Releases del repositorio.
2. Confirmar que existe `JFM PDF Editor Setup 0.1.0.exe` junto con `latest.yml` (necesario para el auto-update).
3. Descargar, instalar y abrir un PDF.

---

## Verificación final del hito

- [ ] `npm test` — unitarias en verde.
- [ ] `npx playwright test` — E2E en verde, `crossOriginIsolated === true`.
- [ ] Instalador firmado: `signtool verify /pa` reporta `Successfully verified`.
- [ ] Doble clic sobre un `.pdf` abre JFM PDF Editor.
- [ ] Editar un PDF y guardar sobrescribe el archivo original, no crea copia en Descargas.
- [ ] Sin peticiones de red en la pestaña Network salvo la comprobación de actualizaciones.
- [ ] Publicar `v0.1.1` y confirmar que una instalación de `v0.1.0` ofrece actualizarse.
