# Tutorial 9 Salmon

1. #### **load conda**

```bash
module load anaconda3/2023.07
conda init
```

2. #### **install software**

`````bash
conda install -c bioconda salmon
`````

3. #### **download the transcriptome:**

```bash
curl ftp://ftp.ensemblgenomes.org/pub/plants/release-28/fasta/arabidopsis_thaliana/cdna/Arabidopsis_thaliana.TAIR10.28.cdna.all.fa.gz -o athal.fa.gz
```

Here, we use `curl` command to download the transcriptome from database. 

4. #### **build index**

In this step, we build an index on our transcriptome. The index is a structure that salmon uses to [quasi-map](http://bioinformatics.oxfordjournals.org/content/32/12/i192.abstract) RNA-seq reads during quantification. The index need only be constructed once per transcriptome, and it can then be reused to quantify many experiments. 

```bash
salmon index -t athal.fa.gz -i athal_index
```

Other available options from `salmon index`:

* `--type`: can put `quasi` (default) or `fmd` 
* `-k`: specify the k-mer

This step will create a folder name `athal_index`.

Inside this folder, we can see:

```
athal_index
├── complete_ref_lens.bin
├── ctable.bin
├── ctg_offsets.bin
├── duplicate_clusters.tsv
├── info.json
├── mphf.bin
├── pos.bin
├── pre_indexing.log
├── rank.bin
├── refAccumLengths.bin
├── ref_indexing.log
├── reflengths.bin
├── refseq.bin
├── seq.bin
└── versionInfo.json

0 directories, 15 files
```



5. #### obtaining sequence data

In addition to the *index*, salmon obviously requires the RNA-seq reads from the experiment to perform quantification. In this tutorial, we’ll be analyzing data from [this 4-condition experiment](https://www.ebi.ac.uk/ena/data/view/DRP001761) [accession PRJDB2508]. 

You can use the following shell script to obtain the raw data and place the corresponding read files in the proper locations. 

Here, we’re simply placing all of the data in a directory called `data`, and the left and right reads for each sample in a sub-directory labeled with that sample’s ID (i.e. `DRR016125_1.fastq.gz` and `DRR016125_2.fastq.gz` go in a folder called `data/DRR016125`).

```bash
#!/bin/bash
mkdir data
cd data
for i in `seq 25 40`; 
do 
  mkdir DRR0161${i}; 
  cd DRR0161${i}; 
  wget ftp://ftp.sra.ebi.ac.uk/vol1/fastq/DRR016/DRR0161${i}/DRR0161${i}_1.fastq.gz; 
  wget ftp://ftp.sra.ebi.ac.uk/vol1/fastq/DRR016/DRR0161${i}/DRR0161${i}_2.fastq.gz; 
  cd ..; 
done
cd .. 
```

Place these code in a file called `dl_tut_reads.sh`, than run the script:

`````
bash dl_tut_reads.sh
`````

6. #### Quantifying the samples

```Bash
#!/bin/bash
for fn in data/DRR0161{25..40};
do
samp=`basename ${fn}`
echo "Processing sample ${samp}"
salmon quant -i athal_index -l A \
         -1 ${fn}/${samp}_1.fastq.gz \
         -2 ${fn}/${samp}_2.fastq.gz \
         -p 8 --validateMappings -o quants/${samp}_quant
done 
```

Options for `salmon quant`:

- `-i`: the index we just built
- `-l`: specify library type
  - A: automatically determine the library type
- `-1`/`-2`: tell salmon where to find the left and right reads for this sample
- `-p`: make use of the 8 threads

Other available options:

* `--gcBias` to learn and correct for fragment-level GC biases in the input data
* `--posBias` will enable modeling of a position-specific fragment start distribution

7. #### Salmon output

You should see a new directory has been created that is named by the string value you provided in the `-o` command. In the output folder, there will be a file called `quant.sf`. This is the quantification file in which each row corresponds to a transcript, listed by Ensembl ID. The columns correspond to metrics for each transcript:

`````
Name    Length  EffectiveLength TPM     NumReads
ENST00000632684.1       12      2.000   0.000000        0.000
ENST00000434970.2       9       1.000   0.000000        0.000
ENST00000448914.1       13      2.000   0.000000        0.000
ENST00000415118.1       8       1.000   0.000000        0.000
ENST00000390583.1       31      2.000   0.000000        0.000
ENST00000390577.1       37      2.000   0.000000        0.000
ENST00000451044.1       17      2.000   0.000000        0.000
....
`````

- The first two columns are self-explanatory, the **name** of the transcript and the **length of the transcript** in base pairs (bp).
- The **effective length** represents the various factors that effect the length of transcript (i.e degradation, technical limitations of the sequencing platform)
- Salmon outputs ‘pseudocounts’ or ‘abundance estimates’ which predict the relative abundance of different isoforms in the form of three possible metrics (FPKM, RPKM, and TPM). **TPM (transcripts per million)** is a commonly used normalization method as described in [[1\]](http://www.ncbi.nlm.nih.gov/pmc/articles/PMC2820677/) and is computed based on the effective length of the transcript. **We do NOT recommend FPKM or RPKM**.
- Estimated **number of reads**, which is the estimate of the number of reads drawn from this transcript given the transcript’s relative abundance and length)



#### Reference

https://combine-lab.github.io/salmon/getting_started/

https://mp.weixin.qq.com/s/e7L2fN7CFlKP-bXaoZE_QA
