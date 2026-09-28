# Notas de aprendizaje — NGS Pipeline

Bitácora personal: errores encontrados, por qué pasaron, cómo se resolvieron, y conceptos aprendidos en el camino. Pensado para repasar, no para que lo lea otra persona.

## Setup del entorno

### Conda / Miniconda

- Instalado Miniconda desde el instalador oficial (`Miniconda3-latest-Linux-x86_64.sh`).
- Conda ≠ herramienta de Python: es un gestor de paquetes/entornos genérico. Puede instalar programas en cualquier lenguaje (Java, C++, etc.), no solo librerías Python.
- Cada proyecto debería tener su propio _environment_ aislado, para que las dependencias de un proyecto no choquen con las de otro. Nunca instalar cosas de proyecto directamente en `base`.
- Canales configurados: `defaults`, `bioconda` (herramientas bioinformáticas), `conda-forge`. Con `channel_priority strict` para evitar conflictos de versiones.

### Error: Terms of Service no aceptados

- Al crear el entorno, Conda pidió aceptar ToS de los canales `pkgs/main` y `pkgs/r` (canales comerciales de Anaconda Inc.).
- Solución: `conda tos accept --override-channels --channel <url>` para cada canal.

### sra-tools viejo con error de certificado TLS

- La versión de `sra-tools` que instala Bioconda (2.9.6) tiene certificados TLS desactualizados → falla el handshake HTTPS contra NCBI (`mbedtls_ssl_handshake returned -9984`).
- Intentar `conda update` no sirvió: Conda decía "ya instalado" (probablemente resolviendo a la misma versión vieja disponible en el canal).
- Solución: instalar el SRA Toolkit oficial directo desde NCBI (`sratoolkit.current-ubuntu64.tar.gz`), descomprimirlo, y agregar su carpeta `bin` al PATH en `~/.bashrc`.
- Problema secundario: al tener las DOS versiones instaladas (la de Conda y la nueva), bash encontraba primero la de Conda porque al activar un entorno, Conda se antepone al PATH.
- Solución final: `conda remove -n ngs-pipeline sra-tools -y` para sacar la versión vieja y dejar solo la oficial (3.4.1) en el PATH.

## Conceptos aprendidos

### Single-end vs. paired-end

Un fragmento de ADN puede leerse desde un solo extremo (single-end) o desde ambos extremos (paired-end). Paired-end da más información (mejor precisión de alineamiento, detección de reordenamientos) a costa de generar el doble de datos: dos archivos, `_1.fastq` (forward/R1) y `_2.fastq` (reverse/R2).

### Direccionalidad 5'→3'

Toda secuencia se reporta 5'→3' por convención, siempre. R1 y R2 no son "una al derecho y otra al revés" en el archivo — cada uno está en su propio 5'→3', pero corresponden a hebras opuestas del mismo fragmento, leídas desde extremos opuestos "hacia adentro". Esto se llama orientación FR (forward-reverse), la más común en Illumina paired-end.

## Dataset usado

- Accession: **SRR2584863** (E. coli, cepa REL606, del estudio de evolución experimental de 50.000 generaciones, Lenski lab).
- Mismo accession usado en la lección oficial de Data Carpentry "Wrangling Genomics".
- Descargado con: `fasterq-dump SRR2584863 --split-files --progress`

## FastQC — control de calidad inicial

Corrido con:

```bash
fastqc raw_data/SRR2584863_1.fastq raw_data/SRR2584863_2.fastq -o qc_reports/
```

### Cómo leer un FASTQ a mano

Cada read son 4 líneas: header (`@...`), secuencia, separador (`+...`), calidad.
La línea de calidad tiene el mismo largo que la secuencia, cada carácter corresponde a la base en esa misma posición.
Fórmula: `Phred score = código ASCII del carácter - 33`. Ej: `F` (ASCII 70) → Phred 37 (buena calidad). `#` (ASCII 35) → Phred 2 (calidad pésima).

### Resultados R1 (SRR2584863_1)

- 1,553,259 reads, 150bp cada uno, ~233 Mbp totales.
- Con genoma de E. coli (~4.6 Mbp), esto da ~50x de cobertura teórica solo con R1.
- %GC = 50, coincide con lo esperado para E. coli real (buena señal, no hay indicio de contaminación).
- **Per base sequence quality**: buena calidad en el cuerpo del read, cae progresivamente desde ~posición 100-130 en adelante. Esperado en Illumina (degradación química del reactivo a medida que avanza el ciclo de secuenciación).
- **Per sequence quality scores**: la gran mayoría de los reads tiene calidad promedio muy alta (pico en Phred ~36-37), a pesar de la caída en la cola, el promedio del read completo no se ve muy afectado.
- **Adapter Content** (⚠️): aparece adaptador Nextera Transposase hacia el final del read (~10% en la posición 150). Coincide con la caída de calidad en la misma zona — son fragmentos más cortos que 150bp, así que el secuenciador termina leyendo hacia el adaptador.
- **Per base sequence content** (⚠️) y **Per sequence GC content** (⚠️): ambos son warnings esperados/benignos.
  - El primero se dispara por el sesgo de "priming aleatorio" en las primeras ~15 bases del read, artefacto técnico bien conocido de Illumina, no un problema real (se estabiliza limpiamente después).
  - El segundo es solo una campana de GC levemente más angosta que la teórica, sin picos secundarios, no indica contaminación.

### Resultados R2 (SRR2584863_2) — comparación con R1

R2 tiene más métricas en rojo (❌) que R1:

- Per base sequence quality: ❌ (vs ✅ en R1)
- Per tile sequence quality: ❌ (vs ✅ en R1)
- Per base sequence content: ❌ (vs ⚠️ en R1)

**Por qué R2 es sistemáticamente peor que R1**: es un patrón conocido y esperado en Illumina paired-end. R2 se secuencia después que R1 en la misma corrida, así que los reactivos químicos ya están más degradados para cuando le toca el turno, esto pasa en prácticamente cualquier dataset paired-end de Illumina, no es específico de este dataset.

### Conclusión del QC inicial

Dataset sano en general. Problema real y esperado: calidad decayendo + algo de adaptador Nextera en la cola de los reads (más marcado en R2). Los demás warnings son artefactos técnicos comunes, no problemas genuinos. Esto justifica el siguiente paso: trimming con fastp.

## fastp — trimming

Corrido con:

```bash
fastp \
  -i raw_data/SRR2584863_1.fastq \
  -I raw_data/SRR2584863_2.fastq \
  -o trimmed_data/SRR2584863_1.trimmed.fastq \
  -O trimmed_data/SRR2584863_2.trimmed.fastq \
  --html qc_reports/fastp_report.html \
  --json qc_reports/fastp_report.json
```

### Resultados

- Q30 subió de 89.4%→93.6% (R1) y 73.9%→84.7% (R2) — mejora mucho más marcada en R2, consistente con que R2 partía de peor calidad.
- 502,702 reads descartados por baja calidad (~16% del total). Con ~50x de cobertura original, sigue sobrando cobertura para alinear.
- 319,910 reads tenían adaptador Nextera; se recortaron ~18.7 Mbp de contaminación de adaptador.
- **Insert size peak: 100bp** — el fragmento real de ADN es más corto que los 150bp que lee el secuenciador. Esto explica cuantitativamente por qué había adaptador en la cola de los reads: el secuenciador "se pasa" del fragmento real y termina leyendo hacia el adaptador.

### Verificación con FastQC post-trimming

Corrido FastQC de nuevo sobre `trimmed_data/`. Comparación directa con el reporte pre-trimming:

- **Adapter Content: ⚠️ → ✅** (el problema identificado se resolvió)
- **Per base sequence quality**: la cola ya no cae a zona roja (Phred 2-10); el peor caso ahora se mantiene sobre Phred ~28-30
- **Sequence Length Distribution: ✅ → ⚠️** — esperado: antes todos los reads medían 150bp exacto, ahora hay largos variables porque cada read se recortó según lo que necesitaba. No es un problema real, es el resultado esperado del trimming.

### Conclusión

El trimming resolvió el problema identificado en el QC inicial (adaptador + calidad decayendo en la cola) sin introducir problemas reales nuevos. Datos listos para alineamiento.

## Bowtie2 — alignment

Corrido con:

```bash
bowtie2 -x reference/REL606_index \
  -1 trimmed_data/SRR2584863_1.trimmed.fastq \
  -2 trimmed_data/SRR2584863_2.trimmed.fastq \
  -S alignments/SRR2584863.sam \
  --threads 4 \
  2> alignments/bowtie2_summary.txt
```

### Resultado: 99.46% overall alignment rate

Buen resultado, pero con un detalle que vale la pena entender: solo 38.7% de los pares alinearon "concordantly" (dentro del rango de distancia esperado por Bowtie2), mientras que la gran mayoría del resto alineó "discordantly".

**Por qué pasa esto**: no es un problema de calidad de los datos. Se debe a que el fragmento real de ADN de esta librería es más corto (~100bp, según el insert size peak reportado por fastp) que el largo combinado de R1+R2 (150bp cada uno). Esto hace que ambos mates se solapen fuertemente entre sí, y esa distancia "R1-inicio a R2-fin" cae fuera del rango que Bowtie2 considera "concordante" por default (aunque cada mate individualmente sí alinee correctamente y de forma única). Es un patrón documentado y conocido en la comunidad bioinformática para librerías con fragment size corto relativo al read length, no indica error de secuenciación, contaminación, ni problema del pipeline.

Referencia: mismo patrón discutido en foros de bioinformática (Biostars) para casos análogos con librerías de fragmento corto.

## SAMtools: SAM → BAM, sort, index

Corrido con:
```bash
samtools view -b alignments/SRR2584863.sam > alignments/SRR2584863.bam
samtools sort alignments/SRR2584863.bam -o alignments/SRR2584863.sorted.bam
samtools index alignments/SRR2584863.sorted.bam
```

### Qué hace cada paso y por qué
- **view -b**: SAM (texto plano) → BAM (binario comprimido).
- **sort**: reordena por posición genómica. El SAM viene en el orden de los reads del FASTQ; casi todas las herramientas downstream exigen el BAM ordenado.
- **index**: genera el `.bai`, que permite acceso aleatorio a cualquier región sin leer el archivo entero (lo usan visores como IGV).

### Tamaños (evidencia de la compresión)
- `.sam`: 1.1 GB
- `.bam` sin ordenar: 319 MB
- `.sorted.bam`: 217 MB (más chico que el BAM sin ordenar porque al ordenar, reads vecinos quedan juntos y comprimen mejor)
- `.bai`: 15 KB

### Verificación con samtools flagstat
Los números coinciden con el resumen de Bowtie2, lo que confirma que el BAM está íntegro:
- 2,602,652 reads totales (= 1,301,326 pares × 2)
- 99.46% mapped (igual al overall alignment rate de Bowtie2)
- 38.74% properly paired (igual a los ~38.7% concordantes de Bowtie2)

"Properly paired" en SAMtools es el mismo concepto que "concordante" en Bowtie2, visto desde otra herramienta.

### Cómo leer el resto del flagstat
- `with itself and mate mapped`: 2,578,134. Casi todos los reads alinearon junto con su pareja. Esto confirma que el bajo "properly paired" no es un problema de calidad: ambos mates alinean bien, pero la distancia entre ellos cae fuera del rango que se considera propio (por el fragmento corto de ~100bp).
- `singletons`: 10,593 (0.41%). Reads que alinearon pero cuya pareja no.
- `mate mapped to a different chr`: 0. Esperable con un genoma de un solo cromosoma. En genomas con varios cromosomas, un valor alto sería señal para investigar.
- `duplicates`: 0. Es cero porque no corrimos ningún paso de marcado de duplicados, no porque no existan (fastp había estimado 0.35%).

### Limpieza
Una vez verificado el `.sorted.bam`, se borraron el `.sam` y el `.bam` sin ordenar (~1.4 GB) porque son regenerables desde el índice + los reads trimmeados. También se borraron los FASTQ crudos de `raw_data/`, ya que el comando de descarga está documentado en el README.
