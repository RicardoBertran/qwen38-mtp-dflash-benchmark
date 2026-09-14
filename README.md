# Qwen 3.8 27B: dos líneas, MTP y DFlash 2

**De 40,94 a 70,84 tok/s en una RTX 5070 de 12 GB: +73,03 % en la prueba del short.** Aquí tienes los presets, el prompt original y los pasos para comparar BASE, DFlash 2 y MTP en tu PC con Windows y llama.cpp / Llama UI.

Las dos líneas protagonistas, dentro del preset del modelo:

```ini
spec-type = draft-mtp
spec-draft-n-max = 2
```

Necesitas un GGUF que conserve los pesos MTP y una build compatible. Estas líneas no añaden MTP a un modelo que no lo tenga.

## Los resultados del short

| Modo | Generación (tok/s) | Cambio frente a BASE | VRAM usada (MiB) | VRAM adicional |
| --- | ---: | ---: | ---: | ---: |
| BASE | 40,94 | — | 10.143 | — |
| DFlash 2 | 41,05 | +0,27 % | 11.566 | +1.423 MiB |
| **MTP** | **70,84** | **+73,03 %** | **10.942** | **+799 MiB** |

GPU: **NVIDIA GeForce RTX 5070, 12 GB**. Modelo: **Qwen 3.8 27B**. Más tok/s significa mayor velocidad de generación; los tokens son fragmentos de texto, no necesariamente palabras.

Son las mediciones originales seleccionadas para el short, aportadas por su autor; no son resultados nuevos ejecutados por este repositorio ni medianas de tres pasadas. DFlash quedó prácticamente empatado en esta prueba. Eso no demuestra que no funcione en otros equipos o tareas.

La VRAM corresponde a lecturas de memoria total usada de la GPU con el modelo cargado; puede incluir escritorio y otros procesos. No representa necesariamente el pico ni la memoria exclusiva del modelo. 1.024 MiB = 1 GiB.

## Qué hace cada opción

- **BASE:** el modelo genera sin decodificación especulativa.
- **DFlash 2:** añade un modelo auxiliar entrenado para este Qwen. Propone un bloque de tokens y el modelo principal comprueba qué propuestas acepta. Si el ahorro supera el coste del auxiliar, acelera; también consume memoria adicional.
- **MTP (Multi Token Prediction):** usa las cabezas de predicción del propio modelo para proponer tokens futuros y verificarlos. En el GGUF utilizado estaban incluidas: no hacía falta otro archivo draft. Otras conversiones pueden omitirlas.

`spec-draft-n-max` limita las propuestas por paso; no es la longitud máxima de la respuesta ni un multiplicador de velocidad. En estos presets se usan 7 para DFlash y 2 para MTP. [Documentación de decodificación especulativa](https://github.com/ggml-org/llama.cpp/blob/master/docs/speculative.md).

## Qué se conoce de la prueba

| Ajuste | Valor |
| --- | --- |
| llama.cpp | b10869, paquete Windows x64 CUDA 13.3 identificado en la sesión original |
| Reasoning | desactivado: `reasoning = off` |
| Contexto | `ctx-size = 4096` |
| Peticiones simultáneas | `parallel = 1` |
| Lote | `batch-size = 128` |
| Temperatura | `temp = 0` |
| Otros ajustes | `top-k = 20`, `top-p = 0.95`, `min-p = 0`, `repeat-penalty = 1`, penalizaciones de frecuencia y presencia = 0 |
| Valores globales recuperados del INI | `jinja = true`, `fit = on` |
| Visión | prueba de texto, sin archivo `mmproj` en los presets |
| Draft DFlash 2 | `Qwen3.8-27B-DFlash2-Q4_K_M.gguf` |

**Límite de reproducibilidad:** no se ha identificado públicamente la cuantización exacta ni la procedencia del GGUF principal usado en el short. Q4_K_M se refiere al **draft DFlash**, no al modelo principal. Tampoco quedaron fijados todos los detalles de driver, reparto CPU/GPU, caché y límite efectivo de salida. El contexto original sugería 768 tokens si la UI lo permitía, pero reportó salidas de longitudes distintas, incluso superiores. Por eso no se presenta 768 como un ajuste confirmado.

Puedes repetir el procedimiento y comparar tus tres modos, pero no prometer una réplica exacta de las cifras. GPU, cuantización, contexto, build, prompt, longitud de salida y ajuste automático de memoria pueden cambiar el resultado. Con `fit = on` puede variar el reparto CPU/GPU entre modos: anótalo. Los resultados no son una garantía universal.

## 1. Instalar o comprobar llama.cpp

Si ya tienes llama-server y Llama UI, conserva tu instalación y comprueba primero las opciones. Para empezar desde cero:

1. Descarga el paquete Windows x64 CUDA de [llama.cpp b10869](https://github.com/ggml-org/llama.cpp/releases/tag/b10869), la versión identificada en la prueba. Si no está disponible, usa una [release oficial](https://github.com/ggml-org/llama.cpp/releases) compatible y registra el cambio.
2. Extrae **todo** el ZIP, incluidas las DLL, en una carpeta propia, por ejemplo `C:\IA\llama`. Si la release distribuye las DLL CUDA por separado, descarga el paquete complementario de esa misma release y extráelo junto al ejecutable.
3. Comprueba que tienes un driver NVIDIA compatible con la versión CUDA del paquete. [Descargas oficiales de NVIDIA](https://www.nvidia.com/Download/index.aspx). El paquete precompilado evita compilar e instalar herramientas de desarrollo.
4. Descarga este repositorio con **Code → Download ZIP**, extrae el ZIP y abre PowerShell en su carpeta. No necesitas instalar Git, Python ni Docker para seguir estos pasos.

Todas las rutas de esta guía son **ejemplos editables**. No necesitas tener unidad E:. Sustituye las rutas por las de tu equipo.

En PowerShell:

```powershell
$llamaServer = 'C:\IA\llama\llama-server.exe'
Test-Path $llamaServer
nvidia-smi
& $llamaServer --version
& $llamaServer --help | Select-String 'models-preset|models-max|draft-mtp|draft-dflash|spec-draft-model|spec-draft-n-max|reasoning|fit'
```

`Test-Path` debe devolver `True`. Deben aparecer `draft-mtp`, `draft-dflash` y los argumentos utilizados. Guarda la versión para comparar tus resultados. Que una opción aparezca en la ayuda no sustituye a comprobar la carga del modelo.

## 2. Preparar el modelo principal

Utiliza **el mismo GGUF de Qwen 3.8 27B** en las tres pruebas. Debe conservar los pesos MTP para poder usar ese modo. Puedes consultar [los GGUF publicados por ggml-org](https://huggingface.co/ggml-org/Qwen3.8-27B-GGUF) y la ficha de la conversión que elijas; esa fuente es una alternativa para comenzar, no una identificación del archivo del short.

Descarga el archivo de cuantización que elijas desde **Files and versions**. Si está dividido en varias partes, descarga todas a la misma carpeta y usa la primera parte como ruta del modelo. Un Qwen 27B Q4 puede superar 12 GB solo en pesos: no presupongas que cabe entero en la 5070. Una cuantización más pequeña o descargar capas a RAM cambia la comparación. Deja RAM suficiente y espacio para los modelos.

Abre estos tres archivos con el Bloc de notas:

- [configs/qwen38-base.ini](configs/qwen38-base.ini)
- [configs/qwen38-dflash.ini](configs/qwen38-dflash.ini)
- [configs/qwen38-mtp.ini](configs/qwen38-mtp.ini)

En los tres, sustituye la línea `m = E:\models\Qwen\CAMBIA_ESTA_RUTA-Qwen3.8-27B.gguf` por la misma ruta absoluta a tu GGUF. No copies rutas de cachés personales ni hashes del vídeo. Guarda los INI en UTF-8.

## 3. Descargar DFlash 2 Q4_K_M

Este es el **modelo auxiliar**, no sustituye al Qwen principal. El archivo publicado ocupa aproximadamente 1,14 GB (1,06 GiB). [Archivo y versiones de Inco AI](https://huggingface.co/incoai/Qwen3.8-27B-DFlash2-GGUF/tree/main).

En PowerShell, cambia la carpeta de ejemplo si no tienes E:. Estos comandos descargan a un archivo temporal y no sobrescriben un draft existente:

```powershell
$draftDir = 'E:\models\DFlash' # EJEMPLO: cambia la unidad/carpeta si hace falta
New-Item -ItemType Directory -Force -Path $draftDir | Out-Null
$draftFile = Join-Path $draftDir 'Qwen3.8-27B-DFlash2-Q4_K_M.gguf'
if (Test-Path $draftFile) { throw 'Ya existe el draft: compruébalo antes de descargar de nuevo.' }
curl.exe --fail --location --retry 3 'https://huggingface.co/incoai/Qwen3.8-27B-DFlash2-GGUF/resolve/main/Qwen3.8-27B-DFlash2-Q4_K_M.gguf?download=true' --output "$draftFile.part"
if ($LASTEXITCODE -ne 0) { throw 'Descarga incompleta. No uses el archivo .part.' }
Move-Item -LiteralPath "$draftFile.part" -Destination $draftFile
Get-Item -LiteralPath $draftFile | Select-Object Name,Length
```

El tamaño sirve para detectar descargas claramente incompletas; no acredita integridad. La URL apunta a `main`, que puede cambiar: para comparaciones rigurosas registra la revisión pública descargada.

En `configs/qwen38-dflash.ini`, conserva estas líneas y adapta únicamente la ruta si descargaste en otro lugar:

```ini
; Ruta de EJEMPLO editable
spec-draft-model = E:\models\DFlash\Qwen3.8-27B-DFlash2-Q4_K_M.gguf
spec-type = draft-dflash
spec-draft-n-max = 7
```

Escribe `Q4_K_M` con guiones bajos normales, sin barras invertidas delante. Usamos archivo local para evitar depender de la resolución automática del repositorio draft al arrancar.

## 4. Arrancar BASE, DFlash o MTP

Desde la carpeta del repositorio, en la misma ventana PowerShell donde definiste `$llamaServer`:

```powershell
& $llamaServer --models-preset '.\configs\qwen38-base.ini' --models-max 1 --host 127.0.0.1 --port 8080
```

Abre **http://127.0.0.1:8080** en tu navegador. Llama UI es aquí la interfaz web servida por llama-server; no hace falta otra aplicación. Selecciona `qwen3.8-27b-SHORT-BASE` y espera a que cargue.

Para cambiar de modo, termina ese servidor con **Ctrl+C**, comprueba que se libera su memoria y ejecuta **uno** de los siguientes comandos:

```powershell
# DFlash 2
& $llamaServer --models-preset '.\configs\qwen38-dflash.ini' --models-max 1 --host 127.0.0.1 --port 8080
```

```powershell
# MTP
& $llamaServer --models-preset '.\configs\qwen38-mtp.ini' --models-max 1 --host 127.0.0.1 --port 8080
```

Recarga la página y selecciona el preset correspondiente. No dejes otro servidor o aplicación de IA con un modelo ocupando la GPU. No añadas un proyector de visión. Los INI contienen todos los ajustes compartidos: BASE fuerza `spec-type = none`; los otros dos activan solo su método.

Si ya usas un router con varios modelos, puedes copiar las secciones `[qwen3.8-27b-SHORT-...]` a tu INI, después de hacer una copia de seguridad. Mantén una sola cabecera `version = 1`, no dupliques nombres de sección y evita parámetros globales o de arranque que sobrescriban la prueba. Para principiantes recomendamos los arranques separados de arriba. [Presets y router de llama.cpp](https://github.com/ggml-org/llama.cpp/blob/master/tools/server/README.md#model-presets).

## 5. Hacer una comparación limpia

1. Carga BASE. Comprueba contexto 4096, un slot, temperatura 0 y reasoning desactivado también en la UI; ajustes guardados del navegador pueden alterar las peticiones.
2. Haz una respuesta de calentamiento: `Responde únicamente con: OK`.
3. Abre un **chat completamente nuevo**, sin instrucciones de sistema personalizadas, adjuntos, herramientas ni historial.
4. Copia todo [prompts/benchmark.txt](prompts/benchmark.txt), el prompt de código recuperado de la conversación original, y espera a que termine. No necesitas tener un access.log: estás midiendo generación de código, no ejecutándolo.
5. Anota velocidad final, tokens generados, VRAM y si la respuesta terminó o alcanzó el límite. Para tus nuevas pruebas fija el mismo máximo de salida en todos los modos; por ejemplo 1.024 tokens. Este es un criterio propuesto para nuevas mediciones, no un valor confirmado del short.
6. Repite tres veces, siempre con chat nuevo. Cambia a DFlash y luego a MTP, con un calentamiento por modo y tres chats nuevos por modo.
7. Compara la **mediana** de cada modo (el número central al ordenar sus tres velocidades). Conserva también las tres mediciones: no elijas solo la mejor.

Nunca ejecutes MTP en una conversación que ya contiene la respuesta de DFlash o BASE. La reutilización de respuestas/contexto altera la prueba. Conserva los mismos límites y configuraciones de caché y registra diferencias en el número de tokens de salida. Las cifras del short no son una comparación de latencia de respuestas idénticas.

### Medir tokens por segundo

Usa la velocidad final de **generación** que muestra Llama UI. En consola, busca el tiempo de generación (`eval time`), no `prompt eval time`, que mide lectura del prompt. Si la build muestra estadísticas específicas de generación especulativa, registra el throughput final de tokens aceptados/salida, no la velocidad del draft ni solo sus pasos internos. Conserva la misma fuente de medición para los tres modos.

No dividas por los segundos redondeados de la interfaz para reconstruir decimales: registra el valor final de tok/s directamente. Carga del modelo, procesamiento del prompt y tiempo hasta el primer token son métricas diferentes.

### Medir VRAM

En otra ventana PowerShell, con el modelo cargado y al acabar la respuesta:

```powershell
nvidia-smi --query-gpu=index,name,memory.used,memory.total --format=csv
```

Para observar el consumo cada segundo durante la generación:

```powershell
nvidia-smi --query-gpu=timestamp,index,memory.used,memory.total,utilization.gpu --format=csv -l 1
```

Termina el seguimiento con Ctrl+C. Si tienes varias GPU, usa siempre el mismo índice. Para repetir el criterio original registra la lectura con el modelo cargado; si además registras el máximo observado durante generación, ponlo en otra columna. El muestreo por segundo puede perder picos breves.

### Plantilla para compartir tu resultado

```text
GPU y VRAM:
CPU y RAM:
Driver NVIDIA:
Build de llama.cpp y variante CUDA:
Repo público / archivo / revisión / cuantización del Qwen principal:
Repo público / archivo / revisión del draft:
Contexto / batch / parallel / reasoning / temperatura:
Límite de salida / configuración de caché / capas en GPU por modo:
Fuente de tok/s: UI o consola (especificar)
BASE:   ___ / ___ / ___ tok/s; mediana ___; VRAM ___ MiB
DFlash: ___ / ___ / ___ tok/s; mediana ___; VRAM ___ MiB
MTP:    ___ / ___ / ___ tok/s; mediana ___; VRAM ___ MiB
Tokens generados por pasada / respuesta truncada / observaciones:
```

Comparte datos técnicos sin rutas de usuario, tokens de acceso ni logs sin revisar. Mejora porcentual = `(velocidad del modo / velocidad BASE - 1) × 100`.

## Si algo falla

| Problema | Qué comprobar |
| --- | --- |
| Opción desconocida | Comprueba `--version` y `--help`; necesitas una build que admita ambos métodos. |
| No aparece el preset | Comprueba ruta del INI, nombre de sección y reinicia el servidor. |
| No encuentra el GGUF | Sustituye el placeholder `CAMBIA_ESTA_RUTA`; usa `Test-Path 'RUTA_REAL'`. |
| Error al cargar DFlash | Revisa descarga completa, ruta sin escapes en `Q4_K_M`, compatibilidad con Qwen 3.8 27B y log final. |
| MTP no carga o no se activa | Comprueba que el GGUF conserva pesos MTP y que el log confirma el método; no todos los GGUF sirven. |
| Falta una DLL CUDA | Extrae las dependencias indicadas por la misma release junto al ejecutable. |
| Error de memoria | Cierra otros modelos; revisa cuantización y reparto CPU/GPU. Si cambias contexto o capas, repite los tres modos y registra el cambio. |
| Puerto 8080 ocupado | Cierra tu servidor anterior o usa 8090 tanto en comando como en navegador. |
| Sigue razonando | Revisa el ajuste de la UI y el preset; `reasoning-format = none` no equivale a desactivar reasoning. |

## Archivos y fuentes

```text
README.md
configs/qwen38-base.ini
configs/qwen38-dflash.ini
configs/qwen38-mtp.ini
prompts/benchmark.txt
```

Fuentes técnicas: [llama.cpp](https://github.com/ggml-org/llama.cpp), [DFlash (proyecto)](https://github.com/z-lab/dflash), [DFlash 2 GGUF](https://huggingface.co/incoai/Qwen3.8-27B-DFlash2-GGUF). Las cifras proceden de la sesión del autor. El repositorio no incluye pesos de modelos, vídeos, rutas personales ni hashes locales.

**¿Vienes del short? Guarda el repositorio, prueba los tres modos y comparte tu GPU y tus tok/s en los comentarios del vídeo.**
