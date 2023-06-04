# Tools
- [bwa v0.17.7](https://bio-bwa.sourceforge.net/)
	- Arm-architecture alternative: [bwa for arm v.0.17.7](https://github.com/smikkelsendk/bwa-for-arm)
- [samtools v1.16.1](https://github.com/samtools/samtools)
- [trimmomatic v0.39](http://www.usadellab.org/cms/?page=trimmomatic)
- [bcftools 1.16](http://www.htslib.org/download/)
- [R lang v4.2.2](https://www.r-project.org/)
- [tassel v5](https://tassel.bitbucket.io/)
- [plink v2.0](https://www.cog-genomics.org/plink/2.0/)
- [Java v11.0.17](https://adoptium.net/temurin/releases/?version=11)
- [MultiQC v1.13](https://multiqc.info/)
	- [FastQC v0.11.9](https://github.com/s-andrews/FastQC/releases)
	- [qualimap v2.2.1](http://qualimap.conesalab.org/)
- Unix based system
# Report 
https://drive.google.com/file/d/1rR11jFDYwYas8jbnMMBHi1BifVHOqqAZ/view?usp=sharing
# Pipeline
# Workflow
## Step 0: Prepare
1. Folder structure:
![[Pasted image 20230114135525.png]]
2. Pre-process data:
![[Pasted image 20230114135726.png]]
- annote_ref:
	- [GCF_001433935.1_IRGSP-1.0_genomic.gtf](https://ftp.ncbi.nlm.nih.gov/genomes/refseq/plant/Oryza_sativa/representative/GCF_001433935.1_IRGSP-1.0/GCF_001433935.1_IRGSP-1.0_genomic.gtf.gz)
- fastq:
	- [IRIC rice database](https://snp-seek.irri.org/_download.zul)
	- Download based on list: [Github - Oryza sativa rice in Viet Nam](https://github.com/Ph1eu/Bioinformatic-Methods-For-Rice-Gene/blob/main/data/PhenotypeData/GrainPropertiesInVietnam.csv)
- seq_ref:
	- [GCF_001433935.1_IRGSP-1.0_genomic.fna](https://ftp.ncbi.nlm.nih.gov/genomes/refseq/plant/Oryza_sativa/representative/GCF_001433935.1_IRGSP-1.0/GCF_001433935.1_IRGSP-1.0_genomic.fna.gz)

## Step 1: Trimming
- Processing data for each folder in fastq folder
```zsh
mkdir -p ./pipeline_executed_data/IRIS_313-11896/trim_reformat
```

```zsh
java -Xmx8G -jar ./process/Trimmomatic-0.39/trimmomatic-0.39.jar PE -threads 8 \
-phred64 ./data/fastq/IRIS_313-11896/IRIS_313-11896_111220_I631_FCC0E5EACXX_L3_RICwdsRSYHSD11-1-IPAAPEK-90_1.fq.gz ./data/fastq/IRIS_313-11896/IRIS_313-11896_111220_I631_FCC0E5EACXX_L3_RICwdsRSYHSD11-1-IPAAPEK-90_2.fq.gz \
./pipeline_executed_data/IRIS_313-11896/trim_reformat/IRIS_313-11896_1.trim_re.pe.fq.gz ./pipeline_executed_data/IRIS_313-11896/trim_reformat/IRIS_313-11896_1.trim_re.se.fq.gz ./pipeline_executed_data/IRIS_313-11896/trim_reformat/IRIS_313-11896_2.trim_re.pe.fq.gz ./pipeline_executed_data/IRIS_313-11896/trim_reformat/IRIS_313-11896_2.trim_re.se.gz \
ILLUMINACLIP:./process/Trimmomatic-0.39/adapters/TruSeq2-PE.fa:2:30:10 LEADING:20 TRAILING:20 SLIDINGWINDOW:4:20 MINLEN:35 TOPHRED33
```

## Step 2: Data pre-processing and Variant discovery
- Check fastq header tag
```zsh
gunzip -c ./data/fastq/IRIS_313-11896/IRIS_313-11896_111220_I631_FCC0E5EACXX_L3_RICwdsRSYHSD11-1-IPAAPEK-90_1.fq.gz > ./data/fastq/IRIS_313-11896/IRIS_313-11896_111220_I631_FCC0E5EACXX_L3_RICwdsRSYHSD11-1-IPAAPEK-90_1.fq
head ./data/fastq/IRIS_313-11896/IRIS_313-11896_111220_I631_FCC0E5EACXX_L3_RICwdsRSYHSD11-1-IPAAPEK-90_1.fq
rm -rf ./data/fastq/IRIS_313-11896/IRIS_313-11896_111220_I631_FCC0E5EACXX_L3_RICwdsRSYHSD11-1-IPAAPEK-90_1.fq
```

- Get this data
![[Pasted image 20230114140904.png]]

- Fill the above data to `@RG` field below, with
	- ID: FCD0LYU.8
	- SM: FCD0LYU.8
	- PU:
	- LB: Check the Excel file
	- PL: ILLUMINA
	- PM: HiSeq2000
- and run:
```zsh
./process/bwa0717_mac_aarch64 mem \
-t 8 \
-R '@RG\tID:FCC0E5E.3\tSM:FCC0E5E.3\tPU:FCC0E5EACXX.3\tLB:ERR617051\tPL:ILLUMINA\tPM:HiSeq2000' \
-M ./data/seq_ref/GCF_001433935.1_IRGSP-1.0_genomic.fna \
./pipeline_executed_data/IRIS_313-11896/trim_reformat/IRIS_313-11896_1.trim_re.pe.fq.gz ./pipeline_executed_data/IRIS_313-11896/trim_reformat/IRIS_313-11896_2.trim_re.pe.fq.gz | \
samtools sort -@ 8 -u -n -o - | \
samtools fixmate -@ 8 -u -m - - | \
samtools sort -@ 8 -u -O bam - | \
samtools markdup -@ 8 -u -r -s - - | \
bcftools mpileup -Ou -d 8000 -f /Users/tl/Desktop/Test/data/seq_ref/GCF_001433935.1_IRGSP-1.0_genomic.fna - | \
bcftools call -mv -Ob -o /Users/tl/Desktop/Test/pipeline_executed_data/IRIS_313-11896/IRIS_313-11896.vcf
```

- index vcf
```zsh
bgzip -ci /Users/tl/Desktop/Test/pipeline_executed_data/IRIS_313-11896/IRIS_313-11896.vcf > /Users/tl/Desktop/Test/pipeline_executed_data/IRIS_313-11896/IRIS_313-11896.vcf.gz

tabix -p vcf /Users/tl/Desktop/Test/pipeline_executed_data/IRIS_313-11896/IRIS_313-11896.vcf.gz
```

- merge vcf
```zsh
bcftools merge snps.txt > merge.vcf
```
