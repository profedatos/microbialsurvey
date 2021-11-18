# Microbial Surveys

En este tutorial analizaremos una red utilizando secuencias de 16S de la tecnología Illumina.

Primero crearemos un directorio en nuestros `/home` y entraremos al mismo:

``` [bash]
 mkdir qiime2
 cd qiime2
 ```
  
 Descargaremos la metadata del tutorial en nuestro computador
 
 Descargar [aquí](https://data.qiime2.org/2021.8/tutorials/moving-pictures/sample_metadata.tsv)
 
Esta metadata viene con algunas celdas inválidas, lo que es común en proyectos grandes de microbial survey.
Para esto instalaremos keemei en `google-chrome`.

Keemei: es un complemento de Google Sheets para validar metadatos de muestras de microbial surveys. La validación de los metadatos de la muestra es importante antes de comenzar cualquier análisis. 
Instalaremos Keemei siguiendo las instrucciones del [sitio web](https://keemei.qiime2.org/) y luego validaremos la metadata.

Ahora descargaremos la data, para esto haremos una carpeta:

```[bash]
mkdir emp-single-end-sequences
cd emp-single-end-sequences
```
dentro de la carpeta descargaremos los barcodes utilizados:
```
wget \
  -O "emp-single-end-sequences/barcodes.fastq.gz" \
  "https://data.qiime2.org/2021.8/tutorials/moving-pictures/emp-single-end-sequences/barcodes.fastq.gz"
``` 
Ahora descargaremos los reads:
```
wget \
  -O "emp-single-end-sequences/sequences.fastq.gz" \
  "https://data.qiime2.org/2021.8/tutorials/moving-pictures/emp-single-end-sequences/sequences.fastq.gz"
```
Todos los datos que se utilizan como entrada a QIIME 2 están en forma de artefactos QIIME 2 (qiime2 artifacts), 
estos contienen información sobre el tipo de datos y la fuente de los datos. Entonces, lo primero que debemos hacer es importar estos archivos de datos de secuencia en un artefacto QIIME 2.

El tipo semántico de este artefacto QIIME 2 es `EMPSingleEndSequences`. Los artefactos `EMPSingleEndSequences` QIIME 2 contienen secuencias que están [multiplexadas](https://www.illumina.com/techniques/sequencing/ngs-library-prep/multiplexing.html), lo que significa que las secuencias aún no se han asignado a las muestras (de ahí la inclusión de los archivos sequence.fastq.gz y barcodes.fastq.gz, donde los barcode.fastq.gz contienen el barcode de cada lectura asociada con cada secuencia en sequence.fastq.gz.) 


```[bash]
qiime tools import \
  --type EMPSingleEndSequences \
  --input-path emp-single-end-sequences \
  --output-path emp-single-end-sequences.qza

```

Para aprender cómo importar datos de secuencia en otros formatos, puede ver un práctico tutorial [aquí](https://docs.qiime2.org/2021.8/tutorials/importing/).


## Demultiplex

Para demultiplexar secuencias necesitamos saber qué secuencia de barcode está asociada con cada muestra. Esta información está contenida en el archivo de metadata. Se pueden ejecutar los siguientes comandos para demultiplexar las secuencias (el comando demux emp-single se refiere al hecho de que estas secuencias tienen un barcode de acuerdo con el protocolo Earth Microbiome Project y son lecturas single-end). El artefacto demux.qza QIIME 2 contendrá las secuencias demultiplexadas.

El segundo resultado (demux-details.qza) presenta los detalles de la corrección de errores de [Golay](https://en.wikipedia.org/wiki/Binary_Golay_code) y no se explorará en este tutorial (puede visualizar estos datos usando la tabulación de metadatos qiime).

```[bash]
qiime demux emp-single \
  --i-seqs emp-single-end-sequences.qza \
  --m-barcodes-file sample-metadata.tsv \
  --m-barcodes-column barcode-sequence \
  --o-per-sample-sequences demux.qza \
  --o-error-correction-details demux-details.qza
```
Después de la demultiplexación, es útil generar un resumen de los resultados. 
Esto le permite determinar cuántas secuencias se obtuvieron por muestra y también obtener un resumen de la distribución de las calidades de la secuencia en cada posición de los datos de la secuencia.

```[bash]
qiime demux summarize \
  --i-data demux.qza \
  --o-visualization demux.qzv
```
Todos los visualizadores QIIME 2 (es decir, los comandos que toman un parámetro --o-visualización) generarán un archivo .qzv. 
Se pueden ver estos archivos con la vista de herramientas de qiime. Este es el comando para ver la primera visualización, 
pero durante el resto del practico veremos la visualización resultante después de ejecutar el visualizador, 
lo que significa que se debe ejecutar la vista de herramientas qiime en el archivo .qzv que se generó.

```[bash]
qiime tools view demux.qzv
```

## Control de calidad de secuencia y construcción de tablas de características (features)

Los plugins de QIIME 2 están disponibles para varios métodos de control de calidad, algunos son DADA2, Deblur y el filtrado básico basado en quality score. 
En este tutorial, el control de calidad se hará con DADA2 y Deblur. Es solo necesario realizar un paso, ud puede elegir el que quiera.  El resultado de ambos métodos será un artefacto QIIME 2 FeatureTable [Frequency], que contiene recuentos (frecuencias) de cada secuencia única en cada muestra en el dataset, y un artefacto FeatureData [Sequence] QIIME 2, que asigna identificadores de características en FeatureTable a las secuencias que representan.





Nota

A medida que trabaje con una o ambas opciones en esta sección, creará artefactos con nombres de archivo que son específicos del método que está ejecutando (por ejemplo, la tabla de características que genera con dada2 denoise-single se llamará table -dada2.qza). Después de crear estos artefactos, cambiará el nombre de los artefactos de una de las dos opciones a nombres de archivo más genéricos (por ejemplo, table.qza). Este proceso de crear un nombre específico para un artefacto y luego cambiarle el nombre solo se realiza para permitirle elegir cuál de las dos opciones le gustaría usar para este paso, y luego completar el tutorial sin volver a prestar atención a esa elección. Es importante tener en cuenta que en este paso, o en cualquier paso de QIIME 2, los nombres de archivo que le está dando a los artefactos o visualizaciones no son importantes.

## Opción 1: DADA2

DADA2 es una canalización para detectar y corregir (cuando sea posible) los datos de secuencia de amplicones de Illumina. Como se implementó en el complemento q2-dada2, este proceso de control de calidad filtrará adicionalmente cualquier lectura phiX (comúnmente presente en los datos de secuencia del gen marcador Illumina) que se identifique en los datos de secuenciación, y filtrará las secuencias quiméricas.

El método dada2 denoise-single requiere dos parámetros que se utilizan en el filtrado de calidad: --p-trim-left m, que recorta las primeras m bases de cada secuencia, y --p-trunc-len n que trunca cada secuencia en posición n. Esto permite al usuario eliminar regiones de baja calidad de las secuencias. Para determinar qué valores pasar para estos dos parámetros, debe revisar la pestaña Gráfica de calidad interactiva en el archivo demux.qzv que fue generado por qiime demux resume arriba.

Pregunta

Según los gráficos que ve en demux.qzv, ¿qué valores elegiría para --p-trunc-len y --p-trim-left en este caso?

En los gráficos de calidad demux.qzv, vemos que la calidad de las bases iniciales parece ser alta, por lo que no recortaremos ninguna base desde el comienzo de las secuencias. La calidad parece disminuir alrededor de la posición 120, por lo que truncaremos nuestras secuencias en 120 bases. Este siguiente comando puede tardar hasta 10 minutos en ejecutarse y es el paso más lento de este tutorial.

qiime dada2 denoise-single \
  --i-demultiplexed-seqs demux.qza \
  --p-trim-left 0 \
  --p-trunc-len 120 \
  --o-representative-sequences rep-seqs-dada2.qza \
  --o-table table-dada2.qza \
  --o-denoising-stats stats-dada2.qza


qiime metadata tabulate \
  --m-input-file stats-dada2.qza \
  --o-visualization stats-dada2.qzv

```[bash]

mv rep-seqs-dada2.qza rep-seqs.qza
mv table-dada2.qza table.qza
``` 

FeatureTable y FeatureData

Una vez finalizado el paso de filtrado de calidad, podemos explorar los datos resultantes mediante comandos que crearán resúmenes visuales de los datos. 
El comando de resumen de la tabla de características le dará información sobre cuántas secuencias están asociadas con cada muestra y con cada característica, un par de histogramas de esas distribuciones y algunas estadísticas de resumen relacionadas. 

El comando `feature-table tabulate-seqs` proporcionará un mapeo de ID de características a secuencias y proporcionará enlaces para BLAST a cada secuencia contra la base de datos NCBI nt. La última visualización será muy útil más adelante en el tutorial, cuando queramos obtener más información sobre características específicas que son importantes en el dataset.

```[bash]
qiime feature-table summarize \
  --i-table table.qza \
  --o-visualization table.qzv \
  --m-sample-metadata-file sample-metadata.tsv
qiime feature-table tabulate-seqs \
  --i-data rep-seqs.qza \
  --o-visualization rep-seqs.qzv
```

### Generar un árbol para análisis de diversidad filogenética¶

QIIME admite varias métricas de diversidad filogenética, incluida la diversidad filogenética de Faith y UniFrac ponderado y no ponderado. Además de los recuentos de características por muestra (es decir, los datos en el artefacto FeatureTable [Frecuencia] QIIME 2), estas métricas requieren un árbol filogenético arraigado que relacione las características entre sí. Esta información se almacenará en un artefacto QIIME 2 Phylogeny [Rooted]. Para generar un árbol filogenético usaremos el pipeline align-to-tree-mafft-fasttree del plugin q2-phylogeny.

Primero, el pipeline usa el programa mafft para realizar una alineamiento múltiple de las secuencias en nuestro FeatureData [Sequence] para crear un artefacto FeatureData del tipo  QIIME 2 [AlignedSequence]. Luego, el pipeline enmascara (o filtra) el alineamiento para eliminar posiciones que son altamente variables.

Generalmente se considera que estas posiciones agregan ruido al árbol filogenético resultante. Después de eso, el pipeline aplica FastTree para generar un árbol filogenético a partir del alineamiento enmascarado. El programa FastTree crea un árbol sin raíz, por lo que en el paso final de esta sección se aplica la raíz del punto medio para colocar la raíz del árbol en el punto medio de la distancia de punta a punta más larga en el árbol sin raíz.

```[bash]
qiime phylogeny align-to-tree-mafft-fasttree \
  --i-sequences rep-seqs.qza \
  --o-alignment aligned-rep-seqs.qza \
  --o-masked-alignment masked-aligned-rep-seqs.qza \
  --o-tree unrooted-tree.qza \
  --o-rooted-tree rooted-tree.qza
```

## Análisis de diversidad alfa y beta

Los análisis de diversidad de QIIME 2 están disponibles a través del plugin q2-diversity, que realiza el cálculo de métricas de diversidad alfa y beta, la aplicación de pruebas estadísticas relacionadas y la generación de visualizaciones interactivas. Primero aplicaremos el método `core-metrics-phylogenetic`, que enrarece (rarefies) una FeatureTable [Frequency] a una profundidad especificada por el usuario, calcula varias métricas de diversidad alfa y beta y genera gráficos de análisis de coordenadas de principales (PCoA) usando Emperor para cada uno de los métricas de diversidad beta. 

Las métricas calculadas por defecto son:

- Diversidad alfa

 - Índice de diversidad de Shannon (una medida cuantitativa de la riqueza de la comunidad)

 - Características observadas (una medida cualitativa de la riqueza de la comunidad)

 - Faith's Phylogenetic Diversity (una medida cualitativa de la riqueza de la comunidad que incorpora relaciones filogenéticas entre las características)

 - Equidad (o Equidad de Pielou; una medida de equidad comunitaria)

-Diversidad beta

 - Distancia de Jaccard (una medida cualitativa de la disimilitud de la comunidad)

 - Distancia de Bray-Curtis (una medida cuantitativa de la disimilitud de la comunidad)

 - Distancia UniFrac no ponderada (una medida cualitativa de la disimilitud de la comunidad que incorpora relaciones filogenéticas entre las características)

 - Distancia UniFrac ponderada (una medida cuantitativa de la disimilitud de la comunidad que incorpora relaciones filogenéticas entre las características)


Un parámetro importante que debe proporcionarse a este script es --p-sampling-depth, que es la profundidad de muestreo uniforme (es decir, rarefacción). 

Debido a que la mayoría de las métricas de diversidad son sensibles a diferentes profundidades de muestreo en diferentes muestras, este script submuestreará aleatoriamente los recuentos de cada muestra hasta el valor proporcionado para este parámetro. 

Por ejemplo, si proporciona --p-sampling-depth 500, este paso submuestreará los recuentos en cada muestra sin reemplazo para que cada muestra en la tabla resultante tenga un recuento total de 500. Si el recuento total de cualquier muestra (s ) son menores que este valor, esas muestras se eliminarán del análisis de diversidad. Elegir este valor es complicado. Se recomienda que haga su elección revisando la información presentada en el archivo table.qzv que se creó anteriormente. Elija un valor lo más alto posible (para que conserve más secuencias por muestra) y, al mismo tiempo, excluya la menor cantidad de muestras posible.


