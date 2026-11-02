# Nanopore bacterial genomics
##  1. Basecall only: keep barcodes on the reads, don't look for them yet
For this we will use dorado (https://github.com/nanoporetech/dorado)
Log in ito the gpu node always to use dorado. dorado only works with gpus
```bash
RAW_DATA_PATH=/PATH/TO/RUN_SEQUENCING/POD5
```
```bash
mkdir -p slurm_jobs 01_dorado
```
Create a slurm job
```bash
#!/bin/bash
#SBATCH --job-name=dorado_basecall
#SBATCH --account=project_2007691
#SBATCH --partition=gpumedium
#SBATCH --time=24:00:00
#SBATCH --nodes=1
#SBATCH --ntasks-per-node=1
#SBATCH --cpus-per-task=72
#SBATCH --gres=gpu:gh200:1
#SBATCH --output=slurm_jobs/dorado_basecaller_%j.out
#SBATCH --error=slurm_jobs/dorado_basecaller_%j.err

module load bio-apps/v202603
module load dorado/2.1.1
dorado basecaller sup ${RAW_DATA_PATH} --no-trim --device cuda:all --emit-fastq > 01_dorado/01_calls.fastq
```
```bash
sbatch 01_dorado.sh
```
## 2. Demux: find the barcodes, trim them, and split into one file per barcode
### Demux using dorado
```bash
dorado demux --kit-name SQK-RBK114-24 \
    --output-dir 01_dorado/demux/ \
    --emit-fastq \
    01_dorado/01_calls.bam
```
### Demux using Barbell (https://github.com/rickbeeloo/barbell)
```bash
export PATH="/projappl/project_2021928/barbell_tykky/bin:$PATH"
mkdir -p 02_barbell
barbell kit -k SQK-RBK114-24 \
    -i 01_dorado/01_calls.fastq \
    -o 02_barbell/ \
    --maximize
```

## 3. QC fastq files
### Using nanoplot
```bash
export PATH="/projappl/project_2021928/qc_assembly_tykky/bin:$PATH"
mkdir -p 03_nanoplot
for fq in 02_barbell/*.fastq*; do
    name=$(basename "$fq")
    echo $name
    NanoPlot -o 03_nanoplot/$name -f png --fastq "$fq" -t 4
done
```
Concatenate all nanoplot files from different genomes in one file
```bash
module load bio-apps
module load py-multiqc/1.28
multiqc 03_nanoplot/ -o 04_multiqc -d -dd 1 --force
```

### Using nanoQC

```bash
mkdir -p 04_nanoQC
export PATH="/projappl/project_2021928/nanoqc_tykky/bin:$PATH"
for fq in path/to/your/demux/folder/*.fastq*; do
    name=$(basename "$fq")
    echo $name
    nanoQC -o 05_nanoQC/$name $fq
done
```
Concatenate all nanoqc reports in one folder

```bash
mkdir -p 05_nanoQC/reports
for fq in 05_nanoQC/B*; do
    name=$(basename "$fq")
    echo $name
    cp $fq/*html 05_nanoQC/reports/$name.html
done
```


## 4. Trimming an filtering
to trim and filter nanopore reads we use chopper

```bash
mkdir -p 06_chopper
export PATH="/projappl/project_2021928/qc_assembly_tykky/bin:$PATH"
for fq in 02_barbell/*.fastq*; do
    name=$(basename "$fq" | cut -d. -f1)
    zcat -f "$fq" | chopper --trim-approach fixed-crop -q 15 -l 500 --headcrop 20 -t 8 \
        | gzip > 06_chopper/$name.fastq.gz
done
```
in this example we trim at the beginning of the reads 20, we remove reads below qscore 15 and we remove reads <lenght 500.

## 5. QC after trimming
```bash
export PATH="/projappl/project_2021928/qc_assembly_tykky/bin:$PATH"
mkdir -p 07_nanoplot

for fq in 06_chopper/*.fastq*; do
    name=$(basename "$fq")
    echo $name
    NanoPlot -o 07_nanoplot/$name -f png --fastq "$fq" -t 4
done
```
Concatenate all nanoplot results using multiqc (https://github.com/multiqc/multiqc)
```bash
module load bio-apps
module load py-multiqc/1.28
multiqc 07_nanoplot/ -o 08_multiqc -d -dd 1 --force
```

```bash
mkdir -p 08_nanoQC
export PATH="/projappl/project_2021928/nanoqc_tykky/bin:$PATH"
for fq in 06_chopper/*.fastq*; do
    name=$(basename "$fq")
    echo $name
    nanoQC -o 08_nanoQC/$name $fq
done
```

Concatenate all nanoQC files in one folder
```bash
mkdir -p 08_nanoQC/reports
for fq in 08_nanoQC/B*; do
    name=$(basename "$fq")
    echo $name
    cp $fq/*html 08_nanoQC/reports/$name.html
done
```

## 6. Assemble the genomes
For this, we use autocycler (https://github.com/rrwick/Autocycler).
other options (https://github.com/mikolmogorov/Flye)

```bash
#!/bin/bash
#SBATCH --account=project_2007691
#SBATCH --partition=small
#SBATCH --time=24:00:00
#SBATCH --cpus-per-task=16
#SBATCH --mem=48G
#SBATCH --array=1
#SBATCH --output=slurm_jobs/flye_basecaller_%j.out
#SBATCH --error=slurm_jobs/flye_basecaller_%j.err
export PATH="/projappl/project_2021928/autocycler_tykky/bin:$PATH"

mkdir -p 09_flye 09_flye/BC01 && cd 09_flye/BC01

flye \
    --nano-raw # your trimmed reads here \
    --threads 6 \
    --out-dir flye_out
```

