[![Gitpod ready-to-code](https://gitpod.io/button/open-in-gitpod.svg)](https://gitpod.io/#https://github.com/renatopuga/somatico)

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/drive/1r4LDUiQqirUFQT2nIrhVW_AHLaBhmS7y?usp=sharing)


# Análise Somática
GATK 4 Mutect2 Somático

# Aula Prática

![image](https://user-images.githubusercontent.com/8321336/130251648-7ad77cae-435f-49be-950f-b7af5f59fd7a.png)

## Pipeline

* GATK4 - Mutect2
* Gene JAK2
* Referência chr9
  * Sobre as versões do Genoma Humano: https://gatk.broadinstitute.org/hc/en-us/articles/360035890711-GRCh37-hg19-b37-humanG1Kv37-Human-Reference-Discrepancies#grch37
* Amostras: 
  * WP043 (tumor)
  * WP044 (normal)
* https://gatk.broadinstitute.org/hc/en-us/articles/360035894731-Somatic-short-variant-discovery-SNVs-Indels-


## Amostras Extras

- WP190 (tumor) e WP191 (normal)
- WP017 (tumor) e WP018 (normal)

**Nota 1:** Utilizar o af-gnomad chr13 e chr19. 

**Nota 2:** Será preciso baixar o chr13 e chr19 da UCSC.

**Download chr19**
```bash
wget -c https://hgdownload.soe.ucsc.edu/goldenPath/hg19/chromosomes/chr19.fa.gz
```

**Download chr13**
```bash
wget -c https://hgdownload.soe.ucsc.edu/goldenPath/hg19/chromosomes/chr13.fa.gz
```



**Concatenar os arquivos .fa.gz**
> Dica 1: zcat lê arquivos compactados .gz e zip

```bash
zcat chr13.fa.gz chr19.fa.gz | sed -e "s/chr//g" > hg19.fa
```

**Gerar o index do arquivo hg19.fa**
```bash
samtools faidx hg19.fa
```

 ____________________ 
< Primeiro é o chr9 >
 -------------------- 
        \   ^__^
         \  (oo)\_______
            (__)\       )\/\
                ||----w |
                ||     ||



## Workflow
Os arquivos BAM (tumor e normal) já foram gerados e estão prontos para a chamada de variates (ver Anexo 1). Então, agora vamos fazer download da referência `chr9` e gerar o index com o programa `samtools`.

### Download da Referência - chr9

* Download 

```bash
wget -c https://hgdownload.soe.ucsc.edu/goldenPath/hg19/chromosomes/chr9.fa.gz
```

* Alterar nome do header: DE: >chr9 para >9
> Essa alteração é necessária pois no BAM a referência não tinha `>chr` era apenas `>9`.

```bash
zcat chr9.fa.gz | sed -e "s/chr//g" > chr9.fa
```

* Verificar se alteração foi feita com o comando `head`

```bash
head chr9.fa
```
> O comando `head` lê as 10 primeiras linha de um arquivo texto


## samtools

Samtools is a suite of programs for interacting with high-throughput sequencing data. It consists of three separate repositories:



### samtools install
# atualizar 
 ``` sudo apt update 
````

* samtools install (Ubuntu)

```bash
sudo apt-get install samtools 
```

* samtools install (Docker)

```bash
docker pull biocontainers/samtools
```



### samtools faidx e index

* samtools faidx

```bash
samtools faidx chr9.fa
```


## GATK4

> Version: 4.2.2.0

Genome Analysis Toolkit - Variant Discovery in High-Throughput Sequencing Data. https://gatk.broadinstitute.org/



### GATK4 install

GATK4 install Docker

```bash
docker pull broadinstitute/gatk:4.2.2.0
```



GATK4 install Mac e Linux

* Download

```bash
wget -c https://github.com/broadinstitute/gatk/releases/download/4.2.2.0/gatk-4.2.2.0.zip
```

* Descompactar

```bash
unzip gatk-4.2.2.0.zip 
```

* Testando gatk

```bash
./gatk-4.2.2.0/gatk
```



### GATK4 .dict

```bash
./gatk-4.2.2.0/gatk CreateSequenceDictionary -R chr9.fa -O chr9.dict
```



### GATK4 intervals

```bash
./gatk-4.2.2.0/gatk ScatterIntervalsByNs -R chr9.fa -O chr9.interval_list -OT ACGT
```



## Mutect2

Call somatic SNVs and indels via local assembly of haplotypes



### Mutect2 Tumor e Normal
> O comando: `samtools view -H normal_JAK2.bam` você consegue pegar o campo SM: que contém o ID da amostra normal.

```bash
samtools view -H normal_JAK2.bam | grep RG | cut -f6 | sed -e "s/SM://g"
```


```bash
./gatk-4.2.2.0/gatk Mutect2 \
	-R chr9.fa \
	-I tumor_JAK2.bam \
	-I normal_JAK2.bam \
	-normal WP044 \
	--germline-resource af-only-gnomad-chr9.vcf.gz \
	-O somatic.vcf.gz \
	-L chr9.interval_list
```



## Calcular Contaminação



### GetPileupSummaries

Tabulates pileup metrics for inferring contamination

* GetPileupSummaries Tumor

```bash
./gatk-4.2.2.0/gatk GetPileupSummaries \
	-I tumor_JAK2.bam \
	-V af-only-gnomad-chr9.vcf.gz \
	-L chr9.interval_list \
	-O tumor_JAK2.table
```

* GetPileupSummaries Normal

```bash
./gatk-4.2.2.0/gatk GetPileupSummaries \
	-I normal_JAK2.bam \
	-V af-only-gnomad-chr9.vcf.gz \
	-L chr9.interval_list \
	-O normal_JAK2.table
```



### CalculateContamination

Calculate the fraction of reads coming from cross-sample contamination

```bash
./gatk-4.2.2.0/gatk CalculateContamination \
	-I tumor_JAK2.table \
	-matched normal_JAK2.table \
	-O contamination.table
```



### FilterMutectCalls

Filter somatic SNVs and indels called by Mutect2

```bash
./gatk-4.2.2.0/gatk FilterMutectCalls \
	-R chr9.fa \
	-V somatic.vcf.gz \
	--contamination-table contamination.table \
	-O filtered.vcf.gz
```

## VEP ensembl - Anotação

### VEP install

```bash
docker pull ensemblorg/ensembl-vep
```

Criar o diretorio vep_output
```
mkdir -p vep_output
```

Modificar a permissao do diretorio vep_output
```
chmod 777 vep_output
```

* Aplicar apenas no Google Colab

```bash
mkdir -p vep_output
chmod 777 vep_output
```

# rodar o vep

```bash
bash vep-docker.sh
```
# Confirma criação do arquivo 
ls -lh vep_output

# Para ver os resultados:

head vep_output/filtered.vep.tsv

## Panel of Normal (PoN)


# Panel of Normal (PoN)

GATK Best Practices - Exome PoN


* vcf

```bash
wget -c https://storage.googleapis.com/gatk-best-practices/somatic-b37/Mutect2-exome-panel.vcf
```

* vcf.idx

```bash
wget -c https://storage.googleapis.com/gatk-best-practices/somatic-b37/Mutect2-exome-panel.vcf.idx
```
# Criar dicionário
./gatk-4.2.2.0/gatk CreateSequenceDictionary -R hg19.fa -O hg19.dict

# Criar o indice 
samtools faidx hg19.fa
# Cria os intervalos
./gatk-4.2.2.0/gatk ScatterIntervalsByNs -R hg19.fa -O hg19.interval_list -OT ACGT

# Conferir a referencia 
grep -v "^@" hg19.interval_list | head 

# Mutect
./gatk-4.2.2.0/gatk Mutect2 \
  -R hg19.fa \
  -I tumor_wp190.bam \
  --germline-resource af-only-gnomad-chr13-chr19.vcf.gz \
  --panel-of-normals Mutect2-exome-panel.vcf \
  -L hg19.interval_list \
  -O WP190.somatic.pon.vcf.gz
  

# CalculateContamination somente com o table do tumor (ex.: wp190)
# erro: A USER ERROR has occurred: Encountered an IO exception while reading from tumor_wp190.table. Não existe arquivo nomeada tumor_wp190.table (comando ls -lh *.table )

```./gatk-4.2.2.0/gatk CalculateContamination \
	-I tumor_wp190.table \
	-O WP190.contamination.pon.table
``` 

# script corrigido  CalculateContamination somente com o table do tumor (ex.: wp190)

# ./gatk-4.2.2.0/gatk GetPileupSummaries -I tumor_wp190.bam -V af-only-gnomad-chr13-chr19.vcf.gz -L hg19.interval_list -O tumor_wp190.table 

* FilterMutectCalls

```bash
./gatk-4.2.2.0/gatk FilterMutectCalls \
	-R hg19.fa \
	-V WP190.somatic.pon.vcf.gz \
	--contamination-table WP190.contamination.pon.table \
	-O WP190.filtered.pon.vcf.gz
```

# Quantas variantes passaram no filtro: 3

zcat WP190.filtered.pon.vcf.gz | grep -v "^#" | awk '$7=="PASS"' | wc -l

# Quantas variantes caíram em cada filtro, como germline, weak_evidence ou panel_of_normals:

zcat WP190.filtered.pon.vcf.gz | grep -v "^#" | cut -f7 | sort | uniq -c | sort -nr

# Ver as variantes PASS:

zcat WP190.filtered.pon.vcf.gz | grep -v "^#" | awk '$7=="PASS"' | cut -f1-7 | head



# Anexo 1

### Converter BAM para FASTQ

```bash
samtools view -h -b /Volumes/Seagate\ Expansion\ Drive/data-lpfap10/projects/proadi/exome/bam/WP043/WP043.bam 9:5021937-5126899 | samtools bam2fq -1 tumor_R1.fq -2 tumor_R2.fq - 
```

```bash
samtools view -h -b /Volumes/Seagate\ Expansion\ Drive/data-lpfap10/projects/proadi/exome/bam/WP044/WP044.bam 9:5021937-5126899 | samtools bam2fq -1 normal_R1.fq -2 normal_R2.fq - 
```



### Converter BAM para JAK2 BAM

Aqui estão os comandos que foram utilizados para gerar os BAMs intermediários e como gerar arquivos FASTQs de regiões específicas do seu arquivo BAM completo.
> Essas etapas não precisam ser executadas nesse pipeline


# Anexo 2

Rodar amostras extras pareadas:

* Com par:
```bash
sh run.chr13-chr19.sh tumor_JAK2.bam normal_JAK2.bam WP043 WP044
sh run.chr13-chr19.sh tumor_wp017.bam tumor_wp018.bam WP017 WP018
sh run.chr13-chr19.sh tumor_wp190.bam tumor_wp191.bam WP190 WP191
```

* Com PoN:
```bash
sh run.chr13-chr19.pon.sh tumor_JAK2.bam normal_JAK2.bam WP043 WP044 
sh run.chr13-chr19.pon.sh tumor_wp017.bam tumor_wp018.bam WP017 WP018 
sh run.chr13-chr19.pon.sh tumor_wp190.bam tumor_wp191.bam WP190 WP191 

```
> ANÁLISE COM PARES (tumor + normal)
 # WP017 (tumor) × WP018 (normal)

 Mutect2: chama as variantes, compara tumor com normal
./gatk-4.2.2.0/gatk Mutect2 \
  -R hg19.fa \
  -I tumor_wp017.bam \
  -I tumor_wp018.bam \
  -normal WP018 \
  --germline-resource af-only-gnomad-chr13-chr19.vcf.gz \
  -L hg19.interval_list \
  -O WP017.somatic.vcf.gz

>>TUMORWP017

./gatk-4.2.2.0/gatk GetPileupSummaries \
  -I tumor_wp017.bam \
  -V af-only-gnomad-chr13-chr19.vcf.gz \
  -L hg19.interval_list \
  -O WP017.table

>>NORMAL018

./gatk-4.2.2.0/gatk GetPileupSummaries \
  -I tumor_wp018.bam \
  -V af-only-gnomad-chr13-chr19.vcf.gz \
  -L hg19.interval_list \
  -O WP018.table

>>> Variantes candidatas e head 5 
zgrep -vc '^#' WP017.somatic.vcf.gz
head -n 5 WP017.table WP018.table

>>>Calculatar contaminação

./gatk-4.2.2.0/gatk CalculateContamination \
  -I WP017.table \
  -matched WP018.table \
  -O WP017.contamination.table

>>FilterMutectCalls

./gatk-4.2.2.0/gatk FilterMutectCalls \
  -R hg19.fa \
  -V WP017.somatic.vcf.gz \
  --contamination-table WP017.contamination.table \
  -O WP017.filtered.vcf.gz


#  WP190 (tumor) × WP191 (normal)

>>> Mutect 
./gatk-4.2.2.0/gatk Mutect2 \
  -R hg19.fa \
  -I tumor_wp190.bam \
  -I tumor_wp191.bam \
  -normal WP191 \
  --germline-resource af-only-gnomad-chr13-chr19.vcf.gz \
  -L hg19.interval_list \
  -O WP190.somatic.vcf.gz

>> TUMOR 190 
./gatk-4.2.2.0/gatk GetPileupSummaries \
  -I tumor_wp190.bam \
  -V af-only-gnomad-chr13-chr19.vcf.gz \
  -L hg19.interval_list \
  -O WP190.table

>>> NORMAL 191 
./gatk-4.2.2.0/gatk GetPileupSummaries \
  -I tumor_wp191.bam \
  -V af-only-gnomad-chr13-chr19.vcf.gz \
  -L hg19.interval_list \
  -O WP191.table

>>Calcular contaminação 
./gatk-4.2.2.0/gatk CalculateContamination \
  -I WP190.table \
  -matched WP191.table \
  -O WP190.contamination.table

>>Filtrar 
./gatk-4.2.2.0/gatk FilterMutectCalls \
  -R hg19.fa \
  -V WP190.somatic.vcf.gz \
  --contamination-table WP190.contamination.table \
  -O WP190.filtered.vcf.gz

>Variantes PASS 
zcat WP190.filtered.vcf.gz | awk '!/^#/ {print $7}' | sort | uniq -c

# Análise com PoN (só o tumor)
  
  # WP017
  >Mutect
./gatk-4.2.2.0/gatk Mutect2 \
  -R hg19.fa \
  -I tumor_wp017.bam \
  --germline-resource af-only-gnomad-chr13-chr19.vcf.gz \
  --panel-of-normals Mutect2-exome-panel.vcf \
  -L hg19.interval_list \
  -O WP017.somatic.pon.vcf.gz

> tabela
./gatk-4.2.2.0/gatk GetPileupSummaries \
  -I tumor_wp017.bam \
  -V af-only-gnomad-chr13-chr19.vcf.gz \
  -L hg19.interval_list \
  -O WP017.pon.table

> Calcular contaminação 
./gatk-4.2.2.0/gatk CalculateContamination \
  -I WP017.pon.table \
  -O WP017.contamination.pon.table

> Filtro 
./gatk-4.2.2.0/gatk FilterMutectCalls \
  -R hg19.fa \
  -V WP017.somatic.pon.vcf.gz \
  --contamination-table WP017.contamination.pon.table \
  -O WP017.filtered.pon.vcf.gz

  # WP190
> Mutect
./gatk-4.2.2.0/gatk Mutect2 \
  -R hg19.fa \
  -I tumor_wp190.bam \
  --germline-resource af-only-gnomad-chr13-chr19.vcf.gz \
  --panel-of-normals Mutect2-exome-panel.vcf \
  -L hg19.interval_list \
  -O WP190.somatic.pon.vcf.gz

> tabela
./gatk-4.2.2.0/gatk GetPileupSummaries \
  -I tumor_wp190.bam \
  -V af-only-gnomad-chr13-chr19.vcf.gz \
  -L hg19.interval_list \
  -O WP190.pon.table

>Calcular contaminação
./gatk-4.2.2.0/gatk CalculateContamination \
  -I WP190.pon.table \
  -O WP190.contamination.pon.table

> Filtro ./gatk-4.2.2.0/gatk FilterMutectCalls \
  -R hg19.fa \
  -V WP190.somatic.pon.vcf.gz \
  --contamination-table WP190.contamination.pon.table \
  -O WP190.filtered.pon.vcf.gz


# contar as variantes PASS de cada análise:

for f in WP017.filtered.vcf.gz WP017.filtered.pon.vcf.gz WP190.filtered.vcf.gz WP190.filtered.pon.vcf.gz; do echo "$f: $(zcat $f | grep -v '^#' | awk '$7=="PASS"' | wc -l) PASS"; done


# JAK2
 > Baixar cromossomo 9 - ref hg19 e descompactar 
wget -c https://hgdownload.soe.ucsc.edu/goldenPath/hg19/chromosomes/chr9.fa.gz
zcat chr9.fa.gz | sed -e "s/chr//g" > chr9.fa

> indexar referencia
samtools faidx hg19_chr9.fa

# WP043 × WP044
> cria sequencia 
./gatk-4.2.2.0/gatk CreateSequenceDictionary -R chr9.fa -O chr9.dict

> cria intervalo
./gatk-4.2.2.0/gatk ScatterIntervalsByNs -R chr9.fa -O chr9.interval_list -OT ACGT

> 
./gatk-4.2.2.0/gatk Mutect2 -R chr9.fa -I tumor_JAK2.bam -I normal_JAK2.bam -normal WP044 --germline-resource af-only-gnomad-chr9.vcf.gz -L chr9.interval_list -O WP043.somatic.vcf.gz

> 
./gatk-4.2.2.0/gatk GetPileupSummaries -I normal_JAK2.bam -V af-only-gnomad-chr9.vcf.gz -L chr9.interval_list -O WP044.table

> calcula contaminação 
./gatk-4.2.2.0/gatk CalculateContamination -I WP043.table -matched WP044.table -O WP043.contamination.table

> filtro

./gatk-4.2.2.0/gatk FilterMutectCalls -R chr9.fa -V WP043.somatic.vcf.gz --contamination-table WP043.contamination.table -O WP043.filtered.vcf.gz

# só  tumor WP043
> mutect
./gatk-4.2.2.0/gatk Mutect2 -R chr9.fa -I tumor_JAK2.bam --germline-resource af-only-gnomad-chr9.vcf.gz --panel-of-normals Mutect2-exome-panel.vcf -L chr9.interval_list -O WP043.somatic.pon.vcf.gz

> tabela 
./gatk-4.2.2.0/gatk GetPileupSummaries -I tumor_JAK2.bam -V af-only-gnomad-chr9.vcf.gz -L chr9.interval_list -O WP043.pon.table

> contaminação 
./gatk-4.2.2.0/gatk CalculateContamination -I WP043.pon.table -O WP043.contamination.pon.table

> filtro 
./gatk-4.2.2.0/gatk FilterMutectCalls -R chr9.fa -V WP043.somatic.pon.vcf.gz --contamination-table WP043.contamination.pon.table -O WP043.filtered.pon.vcf.gz

> identificar e comparar variantes pass 
for f in WP043.somatic.vcf.gz WP043.filtered.vcf.gz WP043.somatic.pon.vcf.gz WP043.filtered.pon.vcf.gz; do echo "$f: total $(zgrep -vc '^#' $f) | PASS $(zcat $f | awk -F'\t' '!/^#/ && $7=="PASS"' | wc -l)"; done 

zcat WP043.filtered.vcf.gz | awk -F'\t' '!/^#/ && $7=="PASS" {print $1, $2, $4, $5}' 

zcat WP043.filtered.pon.vcf.gz | awk -F'\t' '!/^#/ && $7=="PASS" {print $1, $2, $4, $5}'



# Resultado

 >> for f in WP017.filtered.vcf.gz WP017.filtered.pon.vcf.gz WP190.filtered.vcf.gz WP190.filtered.pon.vcf.gz WP043.filtered.vcf.gz WP043.filtered.pon.vcf.gz; do echo "$f: $(zcat $f | grep -v '^#' | awk '$7=="PASS"' | wc -l) PASS"; done
 
WP043: 1 variante PASS na análise pareada e 14 na análise com PoN.
WP017: 1 variante PASS em cada pareamento
WP190: 2 variantes PASS na análise pareada e 3 com PoN

**Código Fonte (run.chr13-chr19.pon.sh e run.chr13-chr19.sh)**

```bash

# variaveis fixas
gnomad="af-only-gnomad-chr13-chr19.vcf.gz"
interval="hg19.interval_list"
genome="hg19.fa"

# bam e ids 
tumor=$1
normal=$2
id_tumor=$3
id_normal=$4

# rodando mutect2
./gatk-4.2.2.0/gatk Mutect2 \
	-R $genome \
	-I $tumor \
	-I $normal \
	-normal $id_normal \
	--germline-resource $gnomad \
	-O $id_tumor.somatic.vcf.gz \
	-L $interval

# getPileup tumor
./gatk-4.2.2.0/gatk GetPileupSummaries \
	-I $tumor \
	-V $gnomad \
	-L $interval \
	-O $id_tumor.table

# getPileup normal
./gatk-4.2.2.0/gatk GetPileupSummaries \
	-I $normal \
	-V $gnomad \
	-L $interval \
	-O $id_normal.table

# CalculateContamination
./gatk-4.2.2.0/gatk CalculateContamination \
	-I $id_tumor.table \
	-matched $id_normal.table \
	-O $id_tumor.contamination.table

# FilterMutectCalls
./gatk-4.2.2.0/gatk FilterMutectCalls \
	-R $genome \
	-V $id_tumor.somatic.vcf.gz \
	--contamination-table $id_tumor.contamination.table \
	-O $id_tumor.filtered.vcf.gz

```

Rodar amostras extras com Panel of Normal (broad institute).

```bash

# variaveis fixas
gnomad="af-only-gnomad-chr13-chr19.vcf.gz"
interval="hg19.interval_list"
genome="hg19.fa"
pon="Mutect2-exome-panel.vcf"

# bam e ids 
tumor=$1
id_tumor=$3

# rodando mutect2
./gatk-4.2.2.0/gatk Mutect2 \
	-R $genome \
	-I $tumor \
	--germline-resource $gnomad \
	--panel-of-normals $pon \
	-O $id_tumor.somatic.pon.vcf.gz \
	-L $interval

# getPileup tumor
./gatk-4.2.2.0/gatk GetPileupSummaries \
	-I $tumor \
	-V $gnomad \
	-L $interval \
	-O $id_tumor.pon.table

# CalculateContamination
./gatk-4.2.2.0/gatk CalculateContamination \
	-I $id_tumor.pon.table \
	-O $id_tumor.contamination.pon.table

# FilterMutectCalls
./gatk-4.2.2.0/gatk FilterMutectCalls \
	-R $genome \
	-V $id_tumor.somatic.pon.vcf.gz \
	--contamination-table $id_tumor.contamination.pon.table \
	-O $id_tumor.filtered.pon.vcf.gz
```
