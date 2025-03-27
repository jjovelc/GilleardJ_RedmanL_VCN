# Variant Calling in Nanopore NGS Data

A pipeline for calling variants in worms. Project from John Gilleard's lab

Data associated to this pipeline can be found in the following link, however, only authorized users have access to such data. 

https://uofc-my.sharepoint.com/:f:/r/personal/juan_jovel_ucalgary_ca/Documents/jj/UofC/data/JohnGilleard/variantCallingNanopore?csf=1&web=1&e=oFN8M4

## 1. Description of the project

Machine learning (ML) models for variant calling with Nanopore data have been successfully trained for human, mouse and few other eukaryotic genomes, but such models are ineffective to call variants in worms more so for PCR products harboring antibiotic-induced mutations. Training of ML models on sequences requires large DNA stretches, e.g., full human chromosomes and is not possible with small PCR sequences. To circumvent such limitation, we have built a hybrid bioinformatics pipeline. We initially used the software chopper with the parameter q– 20 (one potential error every 100 bases = 99% accuracy). This results in losing many sequences of lower quality, which usually are needed when sequencing larger targets like genomes but is affordable in our case because of the relatively small size of our target sequence. Such high-quality sequences should have quality scores similar to Illumina sequences and are then processed with mature bioinformatics tools designed for Illumina sequences like GATK. Hereafter, specific bioinformatics procedures used for variant calling in Nanopore data are described.

## 2. Explanation of pipeline

### 2. 1 Installation of pipeline

The bash script checks whether a conda environment called 'call_variants_nano' exists. If it does not, it creates it and recreate environment contained in file `call_variants_nano.yml`.

```bash

# 0. Install conda environment if it does not exist:
ENV_NAME="call_variants_nano"

# Check if the environment exists
conda info --envs | grep -w $ENV_NAME
if [ $? -eq 0 ]; then
    echo "Environment $ENV_NAME already exists."
    # Commands specific to when the environment exists can go here
else
    echo "Environment $ENV_NAME does not exist. Creating..."
    conda env create -f call_variants_nano.yml
fi

# Activate pipeline
eval "$(conda shell.bash hook)"
conda activate call_variants_nano

```

### 2.2 Inspect quality of reads

```bash
# Input variables
QC_THRESHOLD=20
FASTQ_FILES_LIST=$1
FORMAT="${2,,}"
N_READS_THRESHOLD=$3

# 1. Inspect sequence quality of sequences
# Valid extensions are fastq, fq. Compressed files
# fq.gz and fastq.gz are also valid.
while IFS= read -r FILE; do
        pauvre marginplot --fastq $FILE > ${FILE}.stats
done < $FASTQ_FILES_LIST

# Plots and stats will be stored in a directory
# named 
mkdir QC_before_trimming
mv barcode*png QC_before_trimming
mv barcode*stats QC_before_trimming
```

### 2.3 Quality trimming

```bash
while IFS= read -r FILE; do
    case "$FORMAT" in
        fastq|fq)
            chopper -q "$QC_THRESHOLD" -l 100 < "$FILE" > "${FILE}_trim${QC_THRESHOLD}.fq"
            ;;
        gzip|gz)
            zcat "$FILE" | chopper -q "$QC_THRESHOLD" -l 100 > "${FILE}_trim${QC_THRESHOLD}.fq"
            ;;
        *)
            echo "Allowed formats are fastq and gz. Please specify one of these types."
            ;;
    esac
done
```

### 2.4 Re-inspect sequence quality of sequences (once trimmed)

```bash
for FILE in *trim${QC_THRESHOLD}.fq; do pauvre marginplot --fastq $FILE > ${FILE/fq/stats}; done

mkdir QC_after_trimming
mv barcode*png QC_after_trimming
mv barcode*stats QC_after_trimming
```

### 2.5 Select only trimmed files that contain at least $N_READS_THRESHOLD reads

```bash
# Define the directory name
TRIMMED_DIR="trimmed_Q${QC_THRESHOLD}"
# Create a directory for trimmed files
mkdir -p "$TRIMMED_DIR"

# Move trimmed files to the newly created directory
# Assuming 'QC_after_trimming' is a file that also needs to be moved
mv *"_trim${QC_THRESHOLD}.fq" "$TRIMMED_DIR"
mv "QC_after_trimming" "$TRIMMED_DIR"

# Change directory to the one with trimmed files
cd "$TRIMMED_DIR" || exit

# Echo the intention of the following code
echo "Select only trimmed files with at least $N_READS_THRESHOLD reads"

# Create a directory to hold files with at least the specified number of reads
DIR="trim_Q${QC_THRESHOLD}_${N_READS_THRESHOLD}+_fastq"
mkdir -p "$DIR"

# Iterate over trimmed fastq files
for FILE in *"_trim${QC_THRESHOLD}.fq"; do
    if [[ -f "$FILE" ]]; then
        # Get the number of lines in the file
        line_count=$(wc -l < "$FILE")
        # Multiply the number of reads by 4 to get the number of lines
        if (( line_count >= $N_READS_THRESHOLD * 4 )); then
            mv "$FILE" "$DIR/"
            echo "Moved $FILE to $DIR"
        fi
    fi
done

cd trim_Q"${QC_THRESHOLD}"_"${N_READS_THRESHOLD}"+_fastq
```

###  









