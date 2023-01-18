
# DVC: Una herramienta para administrar y versionar experimentos con datos y modelos

Este repositorio demuestra como utilizar DVC para versionar y hacer seguimiento (*tracking*) de datos y modelos para experimentos de machine learning. También veremos como construir pipelines para ejecutar experimentos con un sólo comando. Finalmente veremos como combinar la anterior para realizar y administrar experimentos de ML.

Este es un repositorio plantilla (template), puedes crear una copia personal de este repositorio presionando "Use this template" (botón verde).

La única referencia para este tutorial es la documentación de DVC: https://dvc.org/. La documentación es excelente, por favor revísenla. 

## Ambiente de desarrollo para este tutorial

Para hacer este tutorial se necesita un ambiente de Python con DVC y las librerías usuales de *machine learning*. Si no tiene un ambiente de Python, puede prepararlo rápidamente con:

    conda create -n demo python=3.9 pip scikit-learn pandas
    conda activate demo
    pip install -r requirements.txt

Si ya tiene un ambiente y sólo necesita DVC, puede instalarlo con:

    pip install dvc[gdrive]

Por simpleza del tutorial se utilizará *google drive* como servidor de almacenamiento remoto.

**Nota**: Para otras opciones de servidores remotos vea aquí: https://dvc.org/doc/install/linux#install-with-pip, por ejemplo Amazon S3, Microsoft Azure o un equipo personal que se accede por SSH. Si no está seguro sobre cual remoto utilizar puede instalarlos todos con: `pip install dvc[all]`


## [Empezar a versionar datos con DVC](https://dvc.org/doc/start/data-management/data-versioning)

**Importante:** DVC requiere una carpeta versionada con `git` (el presente repositorio ya está versionado con `git`). Si quisiera usar DVC en un proyecto que no esté versionado, primero debe ejecutar `git init`.

Para inicializar DVC en un repositorio versionado con `git` se utiliza el comando:

    dvc init

Esto creará un directorio oculto `.dvc ` que contiene los archivos de configuración y el directorio `cache` donde se guardarán los binarios que versionaremos.

Para versionar datos se utiliza el comando [`dvc add`](https://dvc.org/doc/command-reference/add). Se puede versionar un archivo individual o un directorio. En el segundo caso, DVC trata el directorio completo como un artefacto individual.

Utilicemos `dvc add` para versionar el directorio `raw_data`

    unzip explorer_ztf_lcs.zip
    dvc add raw_data
    git add .gitignore raw_data.dvc
    git commit -m "raw_data tracked with dvc"
    git push

DVC versiona los artefactos en base a su *hash* MD5, los cuales se guardan en un archivo con extensión `.dvc`. Estos archivos deben ser versionados con `git`.

## Trayendo/enviado datos desde/hacia el cache al remoto

Intente remover un archivo cualquiera dentro del directorio `raw_data` y luego ejecute [`dvc pull`](https://dvc.org/doc/command-reference/pull). Los datos que se han perdido se traen (*pull*) desde el *cache*. Si el *cache* no existe, por ejemplo cuando se clona por primera vez el repositorio, entonces `dvc pull` traerá los datos desde el servidor remoto.

**Nota:** Se puede traer los archivos asociados a un artefacto `.dvc` individual con `dvc pull nombre.dvc`. En cambio `dvc pull` a secas traerá todos los artefactos del repositorio.

Githu y GitLab son servidores remotos para código fuente versionado con `git`. Para respaldar y compartir archivos binarios debemos configurar un servidor remoto para DVC. Los remotos se añaden/configuran/borran con la instrucción [`dvc remote`](https://dvc.org/doc/command-reference/remote#remote)

Para configurar *google drive* como servidor remoto:

    dvc remote add -d mygdrive gdrive://mygdrivefolderID

donde la ID es la secuencia de caracteres que aparece en la URL de la carpeta que queremos utilizar.

Esto crea un remoto por defecto (*default*) en el archivo de configuración `.dvc/config`. Luego, para enviar datos al remoto utilizamos:

    dvc push

La primera vez que ejecutemos este comando en la sesión se abrirá una pestaña en el navegador donde se solicitarán los permisos necesarios para su cuenta de google. 
    
**Nota:** Si tiene problemas con los comandos `push` o `pull` se recomienda agregar el flag `-v` 

**Nota:** Lo anterior es suficiente para uso personal y para una cantidad moderada de archivos. Si desea usar servicios de Google sin tener que aceptar permisos manualmente se recomienda usar *Google Cloud Project* (otros servicios en la nube como Azure ponen menos problemas).


## [Creando un *pipeline* de procesamiento de datos](https://dvc.org/doc/start/data-management/data-pipelines)

Un pipeline de DVC es un archivo `dvc.yaml` que describe las etapas que se ejecutarán secuencialmente, por ejemplo: manipulación de datos, extración de características, entrenamiento de modelo y cálculo de métricas de evaluación. Expresar estos procesos como un pipeline facilita la reproducibilidad y permite la automaticación de trabajos (CI/CD).

Se puede ejecutar el pipeline con el comando:

    dvc repro

Esto creará un archivo `dvc.lock` que debe ser versionado con `git`. Los resultados del pipeline se versionan automáticamente con DVC y se pueden subir al remoto con `dvc push`.

Las etapas (*stages*) del pipeline definen depedencias sobre ciertos *scripts* y artefactos. DVC detecta si hay cambios en estas dependencias para ejecutar sólo las etapas que se requieran. Las etapas se definen con el comando `dvc stage add`.

Los comandos utilizados para crear las etapas en esta demostración fueron:

    dvc stage add -n parse_raw_data -d src/parse_raw_data.py -d raw_data/ -o data python src/parse_raw_data.py 

    dvc stage add -n compute_features -p features.list -d src/compute_features.py -d data/ -o features/ python src/compute_features.py

    dvc stage add -n train_model -p train.max_depth,train.criterion -d src/train_classifier.py -d features/ -o models/ python src/train_classifier.py

    dvc stage add -n evaluate_model -d src/evaluate_classifier.py -d models/ -d features/ -M metrics.json python src/evaluate_classifier.py


Cada flag en `dvc stage add` significa:

- `-n` : nombre de la etapa
- `-d` : dependencias de la etapa (pueden ser varias)
- `-o` : salidas de la etapa (pueden ser varias y serán versionadas por DVC automáticamente)
- `-p` : parámetros de la etapa, deben explicitarse en el archivo `params.yaml`
- `-M` : archivo de métricas (se explica más adelante)

El archivo `params.yaml` puede contener hiperparámetros del modelo, proporciones del dataset, semillas aleatorias y cualquier otro valor que podríamos necesitar cambiar durante nuestra experimentación. 

**Nota:** Puede haber más de un pipeline en un mismo repositorio, pero sólo puede haber unno por directorio. 

## [Métricas](https://dvc.org/doc/start/data-management/metrics-parameters-plots)

La última etapa del pipeline anterior genera un archivo versionado por `git` llamado `metrics.json` 

En este caso `src/evaluate_model.py` calcula el *f1-score* del modelo entrenado y guarda el valor en dicho archivo.

Podemos verificar el desempeño de nuestra evaluación actual con:

    dvc metrics show 

En general queremos mejorar iterativamente nuestros resultados calibrando y comparando nuestros modelos. A continuación veremos como usar DVC para administrar experimentos en base a pipelines y métricas.

**Nota:** Las métricas pueden ser tablas numéricas y también gráficas, se recomienda ver los ejemplos de la documentación para más detalles: https://dvc.org/doc/user-guide/experiment-management/visualizing-plots

## [Experimentos](https://dvc.org/doc/user-guide/experiment-management)

Un experimento en DVC se construye a partir de un pipeline, un archivo de parámetros, y artefactos de datos. 

Los comandos para administrar experimentos empiezan con `dvc exp`. Por ejemplo:

    dvc exp show

presenta una tabla con los resultados de la rama `main` y el *head* actual (`workspace`). La tabla incluye métricas, parámetros y versiones de los artefactos.

Podemos lanzar un experimento con:

    dvc exp run --set-param train.criterion='entropy'

En este caso se modifica el hiperparámetro `criterion` del árbol de decisión al momento de ejecutar. Si volvemos a hacer `dvc exp show` veremos un experimento bajo la rama `main`.

Podemos encolar experimentos agregando el flag `--queue` en `dvc exp run`, luego podemos ejecutar:

    dvc exp run --run-all --jobs P

donde `P` es el número de experimentos encolados que queremos ejecutar en paralelo.

Opcionalmente también podemos administrar los experimentos encolados con los comandos

    dvc queue start/stop/kill/status/logs

**Importante** Los experimentos no se versionan por defecto ya que no están pensados para ser permanentes. Si queremos actualizar nuestro *workspace* con un experimento en particular se utiliza `dvc exp apply` con la ID del experimento. También podemos crear una rama permanente para un experimento con `dvc exp branch`. Finalmente podemos limpiar los experimentos con `dvc exp gc`. 

**Nota:** Si necesitamos respaldar o compartir experimentos (resultados intermedios) existe `dvc exp pull/push`.
