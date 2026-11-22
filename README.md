# Actualización automática de lista M3U

Este repositorio mantiene actualizada automáticamente una lista M3U mediante GitHub Actions.

El workflow descarga periódicamente una lista M3U desde una URL externa, comprueba que el archivo tenga un formato M3U válido y, si detecta cambios respecto a la versión almacenada en el repositorio, crea un commit y lo publica mediante `git push`.

## Funcionamiento

El workflow realiza los siguientes pasos:

1. Clona el repositorio en el runner de GitHub Actions.
2. Descarga la lista M3U desde la URL configurada.
3. Reintenta la descarga hasta tres veces si se produce un error.
4. Comprueba que el archivo descargado no esté vacío y que contenga la cabecera `#EXTM3U`.
5. Guarda el resultado en el archivo `lista_origen.m3u`.
6. Compara el archivo con la versión existente en el repositorio.
7. Si hay cambios, crea un commit con el mensaje `Actualizar lista M3U` y lo sube a la rama actual.

Si la descarga falla o el archivo no parece una lista M3U válida, el workflow finaliza con error y no modifica el repositorio.

## Ejecuciones

El workflow se ejecuta de dos formas:

- **Automáticamente**, mediante `schedule`.
- **Manualmente**, mediante `workflow_dispatch`, desde la pestaña **Actions** del repositorio.

La programación configurada es:

```yaml
cron: '7,37 9-21 * * *'
```

Esto significa que se ejecuta:

- Todos los días.
- A los minutos `07` y `37` de cada hora.
- Entre las 09:00 y las 21:00 **UTC**.

GitHub Actions interpreta las programaciones cron en UTC, por lo que la hora local puede variar según el huso horario. [17][22]

## Archivos implicados

| Archivo | Descripción |
|---|---|
| `.github/workflows/actualizar-lista-m3u.yml` | Workflow de GitHub Actions que descarga, valida y actualiza la lista. |
| `lista_origen.m3u` | Lista M3U descargada y almacenada en el repositorio. |

## Configuración del workflow

El workflow utiliza los siguientes ajustes principales:

```yaml
permissions:
  contents: write

concurrency:
  group: actualizar-lista-m3u-${{ github.ref }}
  cancel-in-progress: false
```

- `contents: write`: permite al token de GitHub Actions crear commits y hacer `push` en el repositorio.
- `concurrency`: evita que se ejecuten varias instancias del workflow al mismo tiempo para la misma rama.
- `cancel-in-progress: false`: si ya existe una ejecución en curso, las nuevas se ponen en cola en lugar de cancelarla.
- `timeout-minutes: 25`: limita la duración máxima de cada ejecución.

## Reintentos de descarga

La descarga se realiza con `curl` y hasta tres intentos:

| Intento | Ajustes adicionales |
|---:|---|
| 1 | Configuración estándar |
| 2 | Fuerza IPv4 mediante `--ipv4` |
| 3 | Fuerza HTTP/1.1 mediante `--http1.1` |

Entre intentos hay una espera de 15 segundos. Cada descarga tiene un tiempo máximo de conexión de 30 segundos y un tiempo total máximo de 180 segundos.

## Validación de la lista

Antes de guardar o publicar la lista, el workflow comprueba que:

- El archivo descargado no esté vacío.
- El contenido incluya la cabecera `#EXTM3U`.
- Se pueda contar el número de entradas `#EXTINF`.
- Se pueda contar el número de declaraciones `#EXTGRP`.

La validación es básica: no analiza las URLs de los canales, no comprueba su disponibilidad ni valida la integridad criptográfica del archivo descargado.

## Permisos necesarios

El workflow necesita permiso de escritura sobre el contenido del repositorio para poder hacer commit y push:

```yaml
permissions:
  contents: write
```

Asegúrate de que la configuración de Actions del repositorio permita que los workflows lean y escriban contenidos, o de que el permiso `contents: write` esté habilitado para este workflow.

## Ejecución manual

Para ejecutar el workflow manualmente:

1. Abre la pestaña **Actions** del repositorio.
2. Selecciona **Actualizar lista M3U**.
3. Pulsa **Run workflow**.
4. Selecciona la rama deseada y confirma la ejecución.

## Posibles fallos

| Mensaje o situación | Causa probable | Solución |
|---|---|---|
| `No se pudo descargar la lista tras 3 intentos` | La URL no responde, está caída, bloquea GitHub Actions o hay un problema de red. | Comprobar la URL manualmente y revisar los registros del workflow. |
| `La respuesta no es una lista M3U válida` | El servidor devolvió HTML, un error, un archivo vacío o contenido no M3U. | Verificar el contenido de la URL y la validez de la lista. |
| `La lista descargada está vacía` | La descarga produjo un archivo de tamaño cero. | Revisar la disponibilidad de la fuente. |
| `git push` falla | La rama remota cambió durante la ejecución. | Ejecutar de nuevo el workflow o añadir `git pull --rebase` antes del push. |
| No se crea commit | El archivo descargado es idéntico al ya almacenado. | Es el comportamiento esperado. |

## Seguridad y limitaciones

Este workflow descarga contenido de una fuente externa y lo publica en el repositorio si cambia. Por ello, la fiabilidad del repositorio depende de la disponibilidad e integridad de esa fuente.

Recomendaciones adicionales:

- Utilizar una fuente de confianza y, si es posible, verificar el contenido mediante un hash conocido.
- Revisar periódicamente el archivo `lista_origen.m3u` y los commits generados automáticamente.
- Limitar la frecuencia de ejecución si no es necesaria una actualización cada media hora.
- No añadir credenciales, tokens ni información privada directamente en el workflow.
- Considerar restringir la ejecución programada a la rama principal.

## Licencia

Añade aquí la licencia que corresponda a tu repositorio, por ejemplo MIT, Apache-2.0 o GPL-3.0.
