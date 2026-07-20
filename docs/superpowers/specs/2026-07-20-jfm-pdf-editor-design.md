# JFM PDF Editor — Diseño

**Fecha:** 2026-07-20
**Estado:** Aprobado para planificación

## Resumen

Aplicación de escritorio para Windows que empaqueta [BentoPDF](https://github.com/alam00000/bentopdf) como producto instalable de marca JFM, distribuido públicamente bajo AGPL-3.0. Objetivo: sustituir Adobe Acrobat y Nitro PDF en escenarios corporativos habituales, con OCR mejorado respecto a la versión web.

Todo el procesamiento ocurre en la máquina del usuario. No hay servidor, no hay telemetría, ningún documento sale del equipo.

## Contexto

BentoPDF es un sitio estático (Vite + TypeScript) sin backend. Todo el trabajo con PDF ocurre en el navegador vía WebAssembly: `qpdf`, `pdf-lib`, LibreOffice WASM, `wasm-vips`, `tesseract.js`. Esto lo hace ideal para empaquetar como app de escritorio: no hay infraestructura que replicar.

Licencia AGPL-3.0-only. La distribución pública de una versión modificada obliga a publicar el código fuente bajo la misma licencia — aceptado y asumido. La AGPL no otorga derechos sobre la marca "BentoPDF"; de ahí el nombre propio.

## Objetivos

- Instalador de Windows firmado, descargable públicamente.
- Paridad funcional con Acrobat Standard en las tareas habituales.
- OCR mejor que el de la versión web en velocidad y en documentos escaneados.
- Integración con el Explorador de Windows: asociación de archivos, menú contextual, miniaturas.
- Actualización automática desde GitHub Releases.

## No objetivos

- **Impresora virtual "Imprimir a PDF".** Windows 10 y 11 ya incluyen "Microsoft Print to PDF". Duplicarlo exigiría un driver v4 con firma WHQL — el componente más caro del proyecto para replicar algo que el sistema ya regala.
- **Microsoft Store en los hitos 1 y 2.** Es un canal adicional planificado como hito 3; no condiciona el diseño de los dos primeros.
- **Paridad con ABBYY FineReader en reconstrucción de tablas.** Requiere un modelo de análisis de layout, que es otra tecnología. Se reevalúa con usuarios reales.
- macOS y Linux.
- Landing page en jfmtechnology.com — proyecto separado.

## Arquitectura

### Relación con upstream

El repositorio `jfm-pdf-editor` **no contiene código de BentoPDF**. BentoPDF entra como submódulo git anclado a un tag (`v2.8.6` inicialmente) y se construye durante el build; su `dist/` se empaqueta como recurso.

Actualizar BentoPDF = mover el submódulo a un tag nuevo y reconstruir. La personalización se hace con overrides de CSS, `config.json` y sustitución de recursos de marca, sin tocar el código upstream.

Si aparece algo que no se pueda lograr sin modificar código de BentoPDF, se añade una carpeta `patches/` aplicada durante el build. No se hace fork completo: los forks completos dejan de sincronizarse con upstream y se congelan.

### Estructura

```
jfm-pdf-editor/
├── electron/
│   ├── main.ts          # ciclo de vida, ventanas, instancia única
│   ├── preload.ts       # puente IPC acotado (contextBridge)
│   ├── server.ts        # HTTP en loopback sirviendo dist/ con COOP/COEP
│   ├── files.ts         # diálogos nativos, guardado in-place, args de CLI
│   └── updater.ts       # electron-updater ← GitHub Releases
├── native/
│   ├── explorer-command/  # IExplorerCommand (Rust + crate windows)
│   ├── thumbnail/         # IThumbnailProvider (Rust)
│   └── msix/              # manifiesto del sparse package
├── ocr/                 # sidecar: tesseract.exe + tessdata_best
├── branding/            # logo, íconos, CSS override, config.json
├── vendor/bentopdf/     # submódulo git, tag fijo
└── build/               # electron-builder, NSIS, firma
```

### Servidor local (decisión crítica)

El renderer **no** carga con `file://`.

LibreOffice WASM y `wasm-vips` dependen de `SharedArrayBuffer`, que exige aislamiento de origen cruzado: `Cross-Origin-Opener-Policy: same-origin` y `Cross-Origin-Embedder-Policy: require-corp`. Bajo `file://` no se pueden emitir cabeceras y esas herramientas fallan.

`server.ts` levanta un servidor HTTP mínimo en `127.0.0.1` con puerto efímero, sirviendo el `dist/` empaquetado con esas cabeceras. La ventana carga `http://127.0.0.1:<puerto>`. Las cabeceras se portan desde [nginx.conf](https://github.com/alam00000/bentopdf/blob/main/nginx.conf) del upstream, no se deducen.

Alternativa descartada: `protocol.handle` de Electron con esquema privilegiado. Evitaría abrir un puerto, pero el aislamiento de origen cruzado en esquemas custom tiene comportamiento inconsistente entre versiones de Chromium. El loopback es más aburrido y más probado. Escucha solo en la interfaz de loopback; no es accesible desde la red.

### Seguridad del renderer

`contextIsolation: true`, `nodeIntegration: false`, `sandbox: true`.

El preload expone una API estrecha vía `contextBridge`: abrir archivo, guardar archivo, guardar como, ejecutar OCR. Nada más. El renderer ejecuta ~50 dependencias de terceros; ninguna debe poder acceder al sistema de archivos directamente.

### Integración con el sistema de archivos

`files.ts` implementa **guardado in situ**: se abre `contrato.pdf` desde el Escritorio, se edita, Ctrl+S sobrescribe el archivo original. La versión web siempre produce `contrato (1).pdf` en Descargas.

Esta es la diferencia funcional principal entre app y página web, y la razón práctica por la que un usuario deja de abrir Acrobat.

Incluye además: apertura por argumento de línea de comandos, instancia única (un segundo `.pdf` abre pestaña en la ventana existente), y diálogos nativos de archivo.

## Identidad visual

**Se conserva la UI de BentoPDF.** Solo se aplica la paleta y la tipografía de JFM. No hay rediseño de interfaz.

Esta decisión mantiene la arquitectura de wrapper intacta: la marca es una capa de CSS, no una modificación del upstream.

### Por qué no se aplica el sistema editorial de JFM

El sistema editorial (`DESIGN-editorial.md`) está optimizado para páginas de marketing: foto a sangre, display de 84px, radios de 44px, botones de hielo con `backdrop-filter`, gradientes cónicos y tilt 3D por `pointermove`. Funciona porque una portada se recorre una vez.

Un editor de PDF es una herramienta densa de uso prolongado. Aplicarle ese lenguaje traería:

- **Coste de GPU:** decenas de botones con `backdrop-filter` y handlers de `pointermove` compitiendo con el canvas que renderiza el PDF.
- **Contraste no garantizado:** un botón translúcido sobre un documento arbitrario del usuario no tiene fondo controlado.
- **Pérdida de densidad:** display grande y radios amplios consumen el espacio de miniaturas y paneles.

El sistema editorial queda reservado para la landing page (proyecto aparte).

### Estado actual del CSS de BentoPDF

Usa Tailwind 4 con bloques `@theme`, pero los colores están hardcodeados en hex dentro de `src/css/styles.css` (`#111827` fondo, `#1f2937` cards, `#4f46e5` acento) en lugar de tokens semánticos. Implicación: tipografía y paleta de acento se cambian con overrides de `@theme`; los hex sueltos requieren una hoja de override adicional.

### Cambios de marca

| Elemento | BentoPDF | JFM PDF Editor |
|---|---|---|
| Acento | `#4f46e5` (índigo) | `#F6921E` (naranja JFM) vía `@theme --color-indigo-600` + hoja de override |
| Fondo | `#111827` (gris azulado) | `--eh-tinta #0F0F12` |
| Cards | `#1f2937` | Derivado neutro de `--eh-tinta` |
| Tipografía | DM Sans | Plus Jakarta Sans vía `@theme --font-sans` |
| Easing | varios | `--eh-ease: cubic-bezier(0.23,1,0.32,1)` |

**Aeromatics NC no se empaqueta.** Es fuente comercial; embeberla en un ejecutable distribuido públicamente requiere licencia de *app embedding* distinta de la de webfont, y con el repositorio público bajo AGPL el binario quedaría expuesto. Solo aparece rasterizada en el ícono de la aplicación y en la imagen del splash.

### Regla de contraste

`#F6921E` sobre fondo claro da ~2.5:1, por debajo del mínimo WCAG de 4.5:1 para texto. Sobre `--eh-tinta` alcanza ~7:1.

Regla: el naranja se usa en rellenos, bordes y acentos. Nunca como color de texto pequeño sobre fondo claro. En modo claro, el naranja va como fondo de botón con texto blanco encima.

## OCR

### Hallazgo que condiciona el diseño

`tesseract.js` **es** Tesseract compilado a WebAssembly — mismo motor, mismos modelos. Sustituirlo por el binario nativo **no mejora la precisión de forma apreciable**.

El binario nativo sí aporta:
- **Velocidad:** multihilo sin sobrecosto de WASM, del orden de 2-3×.
- **Memoria:** el heap del navegador no soporta escaneos de cientos de páginas a alta resolución; un proceso nativo sí.

La precisión mejora en otros tres frentes:

1. **Modelos `tessdata_best`** para español e inglés. `tesseract.js` usa modelos `fast` para minimizar la descarga web; en un instalador ese límite no existe. Es la mejora con mayor rendimiento por esfuerzo del proyecto.
2. **Preprocesamiento:** enderezado, eliminación de ruido, binarización, normalización de DPI. Un escaneo torcido y sucio es la diferencia entre ~85% y ~97% de acierto. Se implementa con `wasm-vips`, **que ya es dependencia de BentoPDF**.
3. **Modo de segmentación de página** elegido según el tipo de documento, en lugar del predeterminado para todo.

### Diseño

Sidecar con frontera limpia:

- El sidecar (`tesseract.exe` + `tessdata_best` es/en) recibe imágenes preprocesadas y devuelve hOCR.
- El preprocesamiento con `wasm-vips` ocurre en el renderer, antes de invocar el sidecar.
- El ensamblado del PDF con capa de texto invisible se queda en el renderer con `pdf-lib`.
- Archivos intermedios en el directorio temporal de la app, borrados al terminar.

Tesseract es Apache 2.0, compatible con la distribución AGPL.

### Límite reconocido

Esto no reconstruye tablas. Tesseract hace reconocimiento de texto, no análisis de estructura documental — ese es el foso de ABBYY. Una factura escaneada produce texto correcto en orden aproximado, no celdas.

Alcanzarlo requeriría un modelo de layout (PaddleOCR PP-Structure o Surya sobre ONNX Runtime), con 100-300 MB de modelos y un proyecto de integración propio. Se decide con usuarios reales, no ahora.

## Integración nativa de Windows

### IExplorerCommand — menú contextual

Windows 11 no muestra shell extensions clásicos en el menú principal. Se implementa un componente COM `IExplorerCommand` registrado mediante **sparse MSIX package** que apunta al ejecutable instalado por NSIS.

Escrito en Rust con el crate `windows`. Comandos: "Unir con JFM PDF Editor" (selección múltiple) y "Abrir con JFM PDF Editor" (archivo único).

### IThumbnailProvider — miniaturas

Mismo stack, componente separado. Renderiza la primera página como bitmap.

Expectativa calibrada: Windows cachea miniaturas de forma agresiva y Acrobat compite por este registro. Habrá reportes de "no aparecen las miniaturas" que se resuelven limpiando el caché del sistema. Es una limitación conocida de la plataforma.

## Firma de código

**Mecanismo elegido: Azure Trusted Signing.**

### Por qué se firma

1. **SmartScreen.** Sin firma, la primera descarga muestra "Windows protegió su PC — aplicación no reconocida", con el botón de continuar oculto tras "Más información". Para un producto nuevo compitiendo contra Acrobat, es donde se pierde la mayoría de las descargas.
2. **Falsos positivos de antivirus.** Las apps Electron sin firmar son marcadas con frecuencia por heurística; más aún con un sidecar que lanza procesos hijo.
3. **Integridad de las actualizaciones.** `electron-updater` verifica la firma antes de instalar. Sin firma no hay verificación posible en un canal que instala binarios en máquinas ajenas.
4. **Requisito técnico del hito 2.** Windows no registra un paquete MSIX sin firmar. Sin certificado no hay menú contextual del Explorador.

Los puntos 1-3 son de conversión y seguridad; el punto 4 es bloqueante.

### Por qué Azure Trusted Signing y no un certificado EV

- Coste del orden de 10 USD/mes frente a 400-700 USD/año de un EV tradicional.
- Servicio en la nube: permite firmar desde GitHub Actions sin token físico. Desde 2023 los certificados tradicionales exigen almacenamiento en hardware, lo que complica la firma desde CI.
- Otorga reputación de SmartScreen.

**Prerequisito a verificar antes del hito 1:** el servicio exige verificación de la entidad legal y, según la documentación disponible, que la organización tenga una antigüedad mínima (del orden de 3 años). Confirmar elegibilidad de JFM Technology y precio vigente antes de comprometer el calendario. **Plan B:** certificado EV tradicional con HSM en la nube (Azure Key Vault o DigiCert KeyLocker).

### Artefactos a firmar

El ejecutable, el instalador NSIS y el paquete MSIX. El nombre del publisher del MSIX debe coincidir exactamente con el sujeto del certificado — las discrepancias causan fallos de instalación silenciosos.

## Actualizaciones

`electron-updater` contra GitHub Releases del repositorio público. Comprobación al arrancar y luego periódica; descarga en segundo plano; aviso no intrusivo de "reiniciar para actualizar". Actualizaciones diferenciales NSIS para transferir deltas.

**Punto espinoso:** el sparse MSIX no se actualiza mediante el flujo de `electron-updater`. Son dos mecanismos actualizando la misma aplicación. El proceso main verifica al arrancar si la versión del paquete MSIX registrado coincide con la que trae el binario y lo re-registra si difiere.

## Publicación

GitHub Actions sobre runner de Windows: construye BentoPDF desde el submódulo, empaqueta Electron, compila los componentes Rust, firma contra el HSM y publica el release. Disparado por tag.

## Pruebas

- **Unitarias (Vitest):** lógica del proceso main — resolución de rutas, parseo de args de CLI, comparación de versiones del MSIX.
- **Integración (Playwright + Electron):** recorrido crítico — abrir PDF por argumento de CLI, editar, Ctrl+S, verificar que el archivo original se modificó.
- **Humo:** validar que `crossOriginIsolated === true` en el renderer. Si falla, LibreOffice WASM y vips están rotos y medio producto no funciona.
- **Nativo:** manual en VMs de Windows 10 y 11. Automatizar registro COM no compensa.

## Manejo de errores

| Fallo | Comportamiento |
|---|---|
| El servidor local no arranca (antivirus, puerto bloqueado) | Reintentar con otro puerto varias veces; si persiste, mensaje con la causa probable — nunca pantalla en blanco |
| El sidecar de OCR se cuelga o falla | Timeout, matar proceso, devolver el PDF sin capa de texto y avisar — nunca perder el documento |
| Falla la actualización | Silencioso; se registra y se reintenta. Jamás bloquea el uso |

**Sin telemetría.** El producto se posiciona sobre "nada sale de tu máquina"; incluir analítica lo contradiría. Los errores van a un log local que el usuario puede adjuntar voluntariamente.

## Hitos

### Hito 1 — Aplicación distribuible

Wrapper Electron, servidor con COOP/COEP, guardado in situ, asociación `.pdf`, apertura por CLI, marca JFM, instalador NSIS firmado, auto-update, CI.

Resultado: descargable desde la landing y ya sustituye a Acrobat para la mayoría de usuarios.

### Hito 2 — Nativo y OCR

`IExplorerCommand`, `IThumbnailProvider`, sparse MSIX, sidecar de Tesseract con `tessdata_best` y preprocesamiento con vips.

El orden importa: los componentes COM son la parte más propensa a atascarse. Publicar el hito 1 primero evita que el proyecto entero quede sin salir por un handler de miniaturas.

### Hito 3 — Microsoft Store

Segundo canal de distribución, sobre el mismo código base.

**Términos de licencia personalizados (obligatorio).** Los términos estándar de la Store imponen restricciones al usuario incompatibles con la familia GPL, y la AGPL prohíbe añadir restricciones. La Store permite publicar términos propios; ahí va la AGPL. Publicar con los términos por defecto incumpliría la licencia de BentoPDF — y JFM es distribuidor downstream, no titular del copyright. Precedente: VLC, Krita e Inkscape.

**Empaquetado.** Target `appx`/`msix` de electron-builder. La aplicación va como Win32 en full trust (`runFullTrust`), no como UWP en AppContainer, lo que preserva el acceso al sistema de archivos y el guardado in situ.

**Dos variantes de build sobre el mismo código:**

| | Descarga directa | Microsoft Store |
|---|---|---|
| Formato | NSIS | MSIX |
| Firma | Azure Trusted Signing | Microsoft |
| Actualización | `electron-updater` | Store |

`electron-updater` debe quedar desactivado en la variante Store.

**Riesgo a verificar temprano:** el servidor HTTP en loopback. Las restricciones de loopback de Windows aplican a UWP en AppContainer y no deberían afectar a una app en full trust, pero es el tipo de componente que un revisor de certificación puede marcar. Validar con una submission temprana antes de invertir en la variante completa.

**Sinergia con el hito 2:** al producir ya un MSIX, el *sparse package* que el hito 2 necesitaba para registrar `IExplorerCommand` deja de ser un paquete independiente — se declara dentro del MSIX principal. Esto elimina el mecanismo de actualización paralelo descrito en la sección de Actualizaciones y simplifica la parte más frágil del hito 2.

**Orden:** posterior al hito 1. Los ciclos de certificación de la Store son de días y frenarían la estabilización de la primera versión. Se aborda cuando la app sea estable, unificándolo con el trabajo nativo del hito 2.

**Verificar antes de comprometer calendario:** las políticas de la Store cambian con frecuencia. Confirmar en Partner Center los términos personalizados, las capacidades requeridas y el coste de la cuenta de desarrollador.

## Cumplimiento AGPL

- Código fuente del wrapper público en GitHub desde el primer release.
- Aviso de licencia visible en la app (diálogo "Acerca de") y en la landing.
- Enlace al código fuente en el instalador y en la aplicación.
- Atribución a BentoPDF como proyecto original.
