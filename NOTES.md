# Notas de aprendizaje — NGS Pipeline

Bitácora personal: errores encontrados, por qué pasaron, cómo se resolvieron, y conceptos aprendidos en el camino. Pensado para repasar, no para que lo lea otra persona.

## Setup del entorno

### Conda / Miniconda
- Instalado Miniconda desde el instalador oficial (`Miniconda3-latest-Linux-x86_64.sh`).
- Conda ≠ herramienta de Python: es un gestor de paquetes/entornos genérico. Puede instalar programas en cualquier lenguaje (Java, C++, etc.), no solo librerías Python.
- Cada proyecto debería tener su propio *environment* aislado, para que las dependencias de un proyecto no choquen con las de otro. Nunca instalar cosas de proyecto directamente en `base`.
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

- Accession: **SRR2584863** (E. coli, cepa REL606, del estudio de evolución experimental de 50.000 generaciones — Lenski lab).
- Mismo accession usado en la lección oficial de Data Carpentry "Wrangling Genomics".
- Descargado con: `fasterq-dump SRR2584863 --split-files --progress`
