# Nanopore bacterial genomics
##  1. Basecall only: keep barcodes on the reads, don't look for them yet
For this we will use dorado (https://github.com/nanoporetech/dorado)
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


# 1. Basecall only: keep barcodes on the reads, don't look for them yet
dorado basecaller sup ${RAW_DATA_PATH} --no-trim --device cuda:all --emit-fastq > 01_dorado/01_calls.fastq
```
```bash
sbatch 01_dorado.sh
```








export PATH="/projappl/project_2021928/barbell_tykky/bin:$PATH"
mkdir -p 02_barbell
#barbell kit -k SQK-RBK114-24 \
#    -i 01_dorado/01_calls.fastq \
#    -o 02_barbell/ \
#    --maximize



#export PATH="/projappl/project_2021928/qc_assembly_tykky/bin:$PATH"


mkdir -p 03_nanoplot

#for fq in 02_barbell/*.fastq*; do
#    name=$(basename "$fq")
#    echo $name
#    NanoPlot -o 03_nanoplot/$name -f png --fastq "$fq" -t 4
#done
