# JFM PDF Editor

Editor de PDF de escritorio para Windows. Une, divide, comprime, firma, cifra, convierte y aplica OCR a documentos PDF.

**Todo el procesamiento ocurre en tu equipo.** No hay servidor, no hay telemetría, ningún documento sale de tu máquina.

> ⚠️ **En desarrollo.** Todavía no hay versión descargable. El diseño técnico está en
> [`docs/superpowers/specs/`](docs/superpowers/specs/).

## Estado

| Hito | Contenido | Estado |
|---|---|---|
| 1 | App, instalador firmado, asociación `.pdf`, auto-actualización | En desarrollo |
| 2 | Menú contextual del Explorador, miniaturas, OCR nativo | Pendiente |

## Sobre este proyecto

JFM PDF Editor empaqueta [**BentoPDF**](https://github.com/alam00000/bentopdf) como aplicación de escritorio para Windows, con marca propia, integración con el Explorador de archivos y OCR mejorado.

Todo el mérito del motor de PDF corresponde a BentoPDF y a sus contribuidores. Este repositorio contiene únicamente la capa de escritorio: proceso Electron, componentes nativos de Windows, sidecar de OCR e instalador. El código de BentoPDF se incorpora como submódulo, sin modificar.

## Licencia

Distribuido bajo **GNU Affero General Public License v3.0** — ver [LICENSE](LICENSE).

BentoPDF es igualmente AGPL-3.0. Conforme a esta licencia, el código fuente completo de esta aplicación está disponible en este repositorio.

## Desarrollado por

[JFM Technology](https://www.jfmtechnology.com)
