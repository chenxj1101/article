# PyClone说明

## 简介

PyClone is statistical model and software tool designed to infer the prevalence of point mutations in heterogeneous cancer samples. The input data for PyClone consists of a set read counts from a deep sequencing experiment, the copy number of the genomic region containing the mutation and an estimate of tumour content.

Pyclone基于的是MCMC算法.

## 安装

官方推荐使用conda来安装PyClone.为了保证环境的稳定，可为PyClone单独建立一个环境，因为PyClone基于Python2.7:

```bash
conda create --name pyclone python=2
```

激活环境：

```bash
source activate pyclone
```

退出环境：

```bash
source deactivate
```

安装PyClone：

```bash
conda install pyclone -c aroth85
```

## 使用

### 分析命令：

```bash
PyClone run_analysis_pipeline --in_files A.tsv B.tsv C.tsv --working_dir pyclone_analysis
```

PyClone可以分析来自同一个病人的不同样本

### 输入文件：

输入文件为tsv文件，具体格式如下

```tsv
mutation_id	ref_counts	var_counts	normal_cn	minor_cn	major_cn	variant_case	variant_freq	genotype
NA12156:BB:chr2:175263063	3812	14	2	0	2	NA12156	0.0036591740721380033	BB
NA12156:BB:chr1:46500613	3933	42	2	0	2	NA12156	0.010566037735849057	BB
NA12156:BB:chr19:43763059	10352	42	2	0	2	NA12156	0.004040792765056763	BB
NA18507:BB:chr5:180166866	2786	1058	2	0	2	NA18507	0.27523413111342354	BB
NA12878:BB:chr10:17875816	3134	190	2	0	2	NA12878	0.057160048134777375	BB
```

各列说明如下:

- mutation_id - A unique ID to identify the mutation. Good names are thing such a the genomic co-ordinates of the mutation i.e. chr22:12345. Gene names are not good IDs because one gene may have multiple mutations, in which case the ID is not unique and PyClone will fail to run or worse give unexpected results. If you want to include the gene name I suggest adding the genomic coordinates i.e. TP53_chr17:753342.

- ref_counts - The number of reads covering the mutation which contain the reference (genome) allele.

- var_counts - The number of reads covering the mutation which contain the variant allele.

- normal_cn - The copy number of the cells in the normal population. For autosomal chromosomes this will be 2 and for sex chromosomes it could be either 1 or 2. For species besides human other values are possible.

- minor_cn - The minor copy number of the cancer cells. Usually this value will be predicted from WGSS or array data.

- major_cn - The major copy number of the cancer cells. Usually this value will be predicted from WGSS or array data.

>If you do not major and minor copy number information you should set the minor copy number to 0, and the major copy number to the predicted total copy number. If you do this make sure to use the total_copy_number for the --prior flag of the build_mutations_file, setup_analysis and run_analysis_pipeline commands. DO NOT use the parental copy number or major_copy_number information method as it assumes you have knowledge of the minor and major copy number.

> Any additional columns in the tsv file will be ignored so feel free to add additional annotation fields.

### 输出

输出文件夹内包含1个配置文件和3个文件夹，分别如下：

- config.yaml - This file specifies the configuration used for the PyClone analysis.

- plots - Contains all plots from the analysis. There will be two sub-folders clusters/ and loci/ for cluster and locus specific plots respectively.

- tables - This contains the output tables with summarized results for the analysis. There will be two tables clusters.tsv and loci.tsv, for cluster and locus specific information.

- trace - This the raw trace from the MCMC sampling algorithm. Advanced users may wish to work with these files directly for generating plots and summary statistics.

## Tips

因为PyClone调用了matplotlib来作图，由于集群的原因，直接跑PyClone的命令，最后会报一个关于agg的错误。

解决方法是修改Pyclone包里`PathOfPyclone/pyclone/post_process/clusters.py`的文件，在最上方的注释下面一行加入：

```python
import matplotlib
matplotlib.use('agg')
```