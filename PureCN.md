# PureCN测试

## 测试环境

```bash
R version 3.4.3 (2017-11-30)
Platform: x86_64-pc-linux-gnu (64-bit)
Running under: CentOS release 6.5 (Final)

Matrix products: default
BLAS: /p299/user/og06/chenxiangjian1609/software/R/lib64/R/lib/libRblas.so
LAPACK: /p299/user/og06/chenxiangjian1609/software/R/lib64/R/lib/libRlapack.so

R_path: /p299/user/og06/chenxiangjian1609/software/R/bin/R
Rscript_path: /p299/user/og06/chenxiangjian1609/software/R/bin/Rscript
PureCN.R_path: /p299/user/og06/chenxiangjian1609/software/miniconda3/lib/R/library/PureCN/

测试路径： /p299/user/og06/chenxiangjian1609/dev/cnv/test2/pure/test_1206/third
```

## 输入文件

- reference.fa: 基因组文件
- panel9b.bed： 捕获区域panel文件
- OG165910035T1CFD20kx9b1_mutect.vcf： Mutect软件生成的VCF文件
- OG165910035T1CFD20kx9b1.csv： CNVkit生成的cns文件转换成SNP6格式的文件。即6列，分别是Sample，chrom，start，end，probes，log2(seg_mean)

## 测试

### step1 Generate an interval file from a BED file containing target coordinates

```bash
/p299/user/og06/chenxiangjian1609/software/R/bin/Rscript \
/p299/user/og06/chenxiangjian1609/software/miniconda3/lib/R/library/PureCN/extdata/IntervalFile.R \
--infile panel9b.bed --fasta reference.fa --outfile baits_hg19_gcgene2.txt --genome hg19 --offtarget
```

生成的**baits_hg19_gcgene2.txt**用于下一步

### step2 determine purity and ploidy

```bash
/p299/user/og06/chenxiangjian1609/software/R/bin/Rscript \
/p299/user/og06/chenxiangjian1609/software/miniconda3/lib/R/library/PureCN/extdata/PureCN.R \
--out result2 --sampleid OG165910035T1CFD20kx9b1 --segfile OG165910035T1CFD20kx9b1.csv --vcf \
OG165910035T1CFD20kx9b1_mutect.vcf --genome hg19 --gcgene baits_hg19_gcgene2.txt
```

生成的结果文件如下：

```bash
result2_segmentation.pdf
result2.rds
result2.log
result2.csv
result2.pdf
```

在csv结果文件中可见tumor样本的纯度和倍性，如下表所示

| Sampleid | Purity | Ploidy |Sex |Contamination |Flagged |Failed |Curated |Comment |
| ------| ------ | ------ | ------| ------ | ------ |------| ------ | ------ |
| OG165910035T1CFD20kx9b1 | 0.22 | 1.336 |？ | 0 | TRUE |FALSE | FALSE | RARE KARYOTYPE;LOW PURITY |
