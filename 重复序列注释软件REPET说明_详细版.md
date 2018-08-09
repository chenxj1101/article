### 简介
REPET是一个寻找基因组中的重复序列并进行注释和分析的软件。主要分为两个部分  
1. TEdenovo
2. TEannot

**TEdenovo**部分是通过denovo的方式找寻重复序列 
首先使用BLASTER对基因组进行自身比对  
使用RECON、GROUPER、PILER这3个软件进行聚类，并匹配寻找散在重复  
对每一个聚类结果进行多序列比对得到一致性序列  
最后根据TEs的特征对一致性序列进行分类，并去除冗余序列，得到一个TE sequences的库
![TEdenovo](https://urgi.versailles.inra.fr/var/storage/images/media/images/te_denovo/8252-4-eng-GB/TE_DeNovo.png)  

**TEannot**部分是konowledge based的方式找寻重复序列   
首先是使用BLASTER、RepeatMasker、CENSOR寻找基因组中的重复序列（库文件可以是TEdenovo产生的，也可以是其他的） 
然后根据经验值对假阳性匹配进行过滤  
通过TRF、RepeatMasker、MREPS寻找并注释基因组中的SSR  
使用MATCHER将属于同一个TE的片段链接起来  
最后将注释以GFF3或gameXML的形式输出  

![TEannot](https://urgi.versailles.inra.fr/var/storage/images/media/images/te_annot/8256-3-eng-GB/TE_Annot.png)  

### 使用
#### 准备
首先请先创建项目目录，然后将基因组文件（fa格式）链接或复制到目录。  
因为REPET的任务信息以及某些数据是通过Mysql操作的，所以事先需要有Mysql的账号，以及一个能使用的库。然后请以以下格式保存  
>mysql_user  
>mysql_pawd  
>datebase

项目目录里两个文件齐全后，请运行以下脚本，会自动生成REPET所需要的配置文件以及接下来需要运行的任务脚本  
```
python /p299/user/og06/chenxiangjian1609/software/REPET/ready.py -i test.fa -p project -g 55 -d db.cfg
```
##### 参数说明  
-i 表示输入文件，需要是fa格式的  
-p 表示项目名称，REPET要求fa文件要与项目名称一致，所以脚本会进行重命名  
-g 输入文件的大小，以M(兆)为单位  
-d 数据库配置文件，即上面提到的保存Mysql信息的文件  

也可以直接用-h参数查看说明：
```
           -i: the input file, must  fasta file
           -p: the project name
           -g: the size of genome, the unit is MB
           -d: the database config file
           -h: print this help
Example:
           python /p299/user/og06/chenxiangjian1609/software/REPET/ready.py -i test.fa -p project -g 55 -d db.cfg
```
REPET要求基因组文件为标准的fa文件，**即每行的碱基数为60个，所以脚本里调用了Itools进行格式化（不管是否符合标准）**。    
同时还要求fa文件中，每段序列的名称以“>XX_i”的形式命名。名称中**不要出现除“_”以外的特殊符号**。  
运行完毕后会生成这样的一个文件目录结构(文件名称与实际参数有关)
```
|-- db.cfg
|-- genome.fa -> /p299/user/og06/chenxiangjian1609/test_REPET/test3/test.fa
|-- prj_1220.fa
|-- step1_TEdenovo.sh
|-- step2_TEannot.sh
|-- step3_statistic.sh
|-- step4_mask.sh
|-- TEannot
|   |-- change_status.sh
|   |-- clean_jobs.sh
|   |-- prj_1220.fa -> /p299/user/og06/chenxiangjian1609/test_REPET/test_1220/prj_1220.fa
|   |-- repbase20.05_aaSeq_cleaned_TE.fa
|   |-- repbase20.05_ntSeq_cleaned_TE.fa
|   |-- run_TEannot.sh
|   |-- setEnv.sh
|   `-- TEannot.cfg
`-- TEdenovo
    |-- change_status.sh
    |-- clean_jobs.sh
    |-- prj_1220.fa -> /p299/user/og06/chenxiangjian1609/test_REPET/test_1220/prj_1220.fa
    |-- run_TEdenovo.sh
    |-- setEnv.sh
    `-- TEdenovo.cfg
```

####  运行
接下来依次运行项目目录下的4个shell脚本即可。
建议每一步都使用nohup运行，如：  
`nohup sh ./step4_mask.sh &`
#### TEdenovo
TEdenovo分为8个步骤，每一步的作用及产生的文件如下  
##### step1: Genomic sequences are cut into batches  
这一步是将基因组分成一定数量的chunk，默认是分成200k一个chunk,每个batch之间有10k的overlap，考虑序列长度的因素，有一部分chunk的长度是低于200K的。然后将一定数量的chunk合成一个batch，多少个chunk合成一个batch，由配置文件TEdenovo.cfg中的 min_nb_seq_per_batch参数决定。水稻测试时，这个参数为25,所以每个batch的大小约为200k*25=5M，batch80(最后一个)由于chuank数量不足，大小约1.8M。  
step1运行完毕会生成 **REPET_rice_db** 文件夹：  
```sh
|-- batches  #存放batch_*.fa文件的文件夹
|-- REPET_rice_chunks.fa  #原始基因组分成chunk后，将所有chunk合成一个新基因组文件，由于chunk有重叠，所以比原始文件大
`-- REPET_rice_chunks.map  #chunk的map文件，说明每个chuank的来源
1 directory, 2 files
```   

##### step2: The genome is aligned to itself using Blast  
第二步是将step1中的每个batch与由chunk合成的基因组进行自身比对寻找HSPs（high-scoring segment pairs），使用的是REPET自己封装的blast。这一步花费的时间较长，对于大基因组来说占用的内存也较大。  
step2运行完毕会生成  **REPET_rice_Blaster** 文件夹  
```sh
|-- REPET_rice.align.not_over.filtered  #保存HSPs的文件
`-- TEdenovo.cfg #调用的配置文件
``` 

##### step2 structural: LTRs retrotransposons are searched in each batch using LTRharves 
这一步是使用LTRharves软件从每个barch中寻找LTR转座子, Step 2 和 step 2 'structural'是独立的，不存在什么关系  
step2 structural运行完毕会生成 **REPET_rice_LTRharvest** 文件夹，里面包含了每个batch的LTRharves结果，以fa文件的形式保存，结果较多，这里不作展示  

##### step3 The repetitives HSP from BLAST are clustered by Recon, Grouper and/or Piler  
使用Recon、Grouper、Piler这3个软件对blast出来的HSP进行聚类  
这步运行完毕，每个方法都会生成对应的一个文件夹
```sh
REPET_rice_Blaster_Grouper
REPET_rice_Blaster_Recon
REPET_rice_Blaster_Piler
```  
每个文件夹内均包含一个 **REPET_rice_Blaster_<method>_3elem_20seq.fa**文件，里面含有序列信息，每个序列的名称表明了它所在的分类  

##### Step 3 structural : The predictions from LTRharvest are clustered using Blastclust or MCL  
使用blastclust或者MCL对LTRharvest的结果进行聚类，这一步的作用于上一步相同，生成的文件也是类似的，同样包含 **REPET_rice_Blaster_<method>_3elem_20seq.fa**文件 

##### Step 4 : A multiple alignment is computed for each cluster, and a consensus sequence is derived from each multiple alignment  
对前面步骤的4个聚类结果进行多序列比对，每一个多序列比对都会有一个consensus sequence。每个聚类方法都会生成一个对应的文件夹，而consensus sequece则保存在相应文件夹内的 **REPET_rice_Blaster_Grouper_Map_consensus.fa**文件内  
```sh
REPET_rice_Blaster_Grouper_Map
REPET_rice_Blaster_Recon_Map
REPET_rice_Blaster_Piler_Map
REPET_rice_LTRharvest_Blastclust_Map
```  
由于进行了多序列比对，所以每个文件夹内都含有非常多的小文件，ls的时候会非常卡，不建议进行此类操作  

##### Step 5 : Particular features are detected on each consensus, such as structural features or homology with known TE, HMM profiles or host genes  
这步其实是使用PASTEClassifier程序的第一步来寻找确定consensus的特征。
setp5运行完毕后会生成 **REPET_rice_Blaster_GrpRecPil_Struct_Map_TEclassif**文件夹，由于里面的文件会被step6用到并修改，所以在step6再说明文件。同时这步也会在mysql数据库里创建其他的一些表，如：REPET_rice_polyA_set、REPET_rice_ORF_map、REPET_rice_chk_TRF_set，后续步骤也会用到。  

##### Step 6 : The consensus are classified using Wicker's classification 
step6是根据step5确定的consensus的特征对consensus进行分类，使用的是REPET自带的PASTEC tool。PASTEC分类是基于Wicker's classification，所以如果分类出来的TEs的命名规则为：  
```s
{Wicker's classification code}_{consensus name}
{Wicker's classification code}-{completeness}_{consensus name}
{Wicker's classification code}-{completeness}-{potential chimeric}_{consensus name}
{Wicker's classification code}-{completeness}-{potential chimeric}_{consensus name}_{reversed}
```  
样例： 
```sh
Complete Copia retrotransposon : RLX-comp_ProjectName-L-B270-Map20
LARD : RXX-LARD_ProjectName-B-R270-Map3
Incomplete TIR: DTX-incomp_ProjectName-B-P350-Map5
Retrotransposon: RXX_ProjectName-L-B28-Map1
TIR potentially chimeric: DTX-comp-chim_ProjectName-B-G78-Map6
Host gene: PotentialHostGene_ProjectName-B-R52-Map3
Not classified: noCat_ProjectName-B-R878-Map8
Not classified re-qualified like a RLX: noCat-RLX-like_ProjectName-B-G143-Map12
Incomplete retrotransposon originally found in negative strand: RLX-incomp_RLX-incomp_ProjectName-L-B230-Map1_reversed
``` 
step6运行完毕后，**REPET_rice_Blaster_GrpRecPil_Struct_Map_TEclassif**里包含2个文件夹：
```
|-- classifConsensus  
|   |-- REPET_rice_Blaster_GrpRecPil_Struct_Map_consensus.fa -> ../detectFeatures/REPET_rice_Blaster_GrpRecPil_Struct_Map_consensus.fa  #consensus的fa文件
|   |-- REPET_rice.classif  #重复序列分类结果文件
|   |-- REPET_rice.classif_stats.txt #重复序列分类结果统计文件
|   |-- REPET_rice_denovoLibTEs.fa -> REPET_rice_withoutRedundancy_negStrandReversed_WickerH.fa #denovo方式最后得到的TE library
|   |-- REPET_rice_withoutRedundancy.classif  #去冗余的分类结果文件
|   |-- REPET_rice_withoutRedundancy.classif_stats.txt  #去冗余的分类结果统计文件
|   |-- REPET_rice_withoutRedundancy.fa  #去冗余结果的fa文件
|   |-- REPET_rice_withoutRedundancy_negStrandReversed.classif   #去冗余并负义链反转的分类结果文件
|   |-- REPET_rice_withoutRedundancy_negStrandReversed.fa  #去容易并负义链反转的fa文件
|   |-- REPET_rice_withoutRedundancy_negStrandReversed_WickerH.classif #以Wicker code开头命名的分类结果文件
|   |-- REPET_rice_withoutRedundancy_negStrandReversed_WickerH.classif_stats.txt #上一个文件的统计结果
|   |-- REPET_rice_withoutRedundancy_negStrandReversed_WickerH.fa #最终结果的fa文件
|   `-- TEdenovo.cfg -> ../../TEdenovo.cfg
`-- detectFeatures
    |-- ORF 
    |   `-- REPET_rice.ORF.map #ORF结果集
    |-- polyA
    |   `-- REPET_rice.polyA.set  # PolyA结果集
    |-- REPET_rice_Blaster_GrpRecPil_Struct_Map_consensus.fa
    |-- SSR
    |   `-- REPET_rice.SSR.set #SSR结果集
    |-- TEdenovo.cfg -> ../../TEdenovo.cfg
    `-- TR
        `-- REPET_rice.TR.set #TR结果集

6 directories, 19 files
```  
生成的classif文件里没有表头，通过查询mysql数据库里的相应的表，各列的表头如下：

seq_name | length | strand | status | class_classif | order_classif | completeness | evidence
---|--- | ---|---|---|---|---|---|---|---|---|---
noCat_REPET_rice-B-G10008-Map3 | 516 | .| ok| noCat| noCat| NA| CI=NA; struct=(SSRCoverage=0.21)
SSR_REPET_rice-B-G10016-Map3 | 623| .| ok| NA| SSR| NA| CI=100; struct=(TElength: >100bps; SSRCoverage=0.77)
DXX-MITE_REPET_rice-B-G10034-Map4| 590| .| ok| II| MITE| NA| CI=20; struct=(TElength: <700bps; TermRepeats: termTIR: 34); other=(TermRepeats: non-termLTR: 115; SSRCoverage=0.00)    


**Wicker's code：**   
```sh
Transposable elements :
	Class I (RXX)
		DIRS (RYX)
		LARD (RXX-LARD)
		LINE (RIX)
		LTR (RLX)
		PLE (RPX)
		SINE (RSX)
		TRIM (RXX-TRIM)

	Class II (DXX)
		Crypton (DYX)
		Helitron (DHX)
		MITE (DXX-MITE)
		Maverick (DMX)
		TIR (DTX)
	noCat (sequence not classified at class AND order levels) 
	XXX (sequence not classified at class level and with potential several orders)

Not transposable elements :
	PotentialHostGene
	rDNA
	SSR
```    

##### Step 7 : SSR and under-represented unclassified consensus are filtered  
这一步是将上一步得到的最后结果进行SSR和低置信度的未分类consensus进行过滤 
运行完毕后生成 **REPET_rice_Blaster_GrpRecPil_Struct_Map_TEclassif_Filtered** 文件夹：
```sh
|-- classifFileFromList.classif  #过滤后的分类文件
|-- classifFileFromList.classif_stats.txt  #过滤后的分类统计文件
|-- REPET_rice_denovoLibTEs_filtered.fa  #过滤后的序列文件
`-- TEdenovo.cfg -> /p299/user/og06/chenxiangjian1609/test_REPET/REPET_rice364/TEdenovo/TEdenovo.cfg
```
文件格式与step6相同

##### Step 8 : The consensus are clustered into families to facilitate manual curation using Blastclust or MCL
使用Blastclust或MCL对denovo consensus进行聚类，每种方法都会生成一个对应的文件夹  
```
REPET_rice_Blaster_GrpRecPil_Struct_Map_TEclassif_Filtered_Blastclust
REPET_rice_Blaster_GrpRecPil_Struct_Map_TEclassif_Filtered_MCL
```  
以REPET_rice_Blaster_GrpRecPil_Struct_Map_TEclassif_Filtered_Blastclust为例：
```sh
|-- REPET_rice_denovoLibTEs_filtered_Blastclust_clusterCons.tab #聚类后表格，每一行代表一个cluster
|-- REPET_rice_denovoLibTEs_filtered_Blastclust.fa  #聚类后的序列文件，序列名称里的Blc1，即代表cluster的名称
|-- REPET_rice_denovoLibTEs_filtered_Blastclust_globalStatsPerCluster.txt #对所有cluster的统计信息
|-- REPET_rice_denovoLibTEs_filtered_Blastclust.statsPerCluster.tab #每个cluster的统计信息
|-- REPET_rice_denovoLibTEs_filtered.fa -> ../REPET_rice_Blaster_GrpRecPil_Struct_Map_TEclassif_Filtered/REPET_rice_denovoLibTEs_filtered.fa 
`-- TEdenovo.cfg -> /p299/user/og06/chenxiangjian1609/test_REPET/REPET_rice364/TEdenovo/TEdenovo.cfg
```  

ste8、step7、step6生成的序列文件均可以作为TEannot的参考TEs library。一般情况还是使用step8的结果文件。  

#### TEannot
TEannot同样分为8个步骤

##### Step 1 genomic sequence and data banks preparation
这一步与TEdenovo的step1类似，也是将基因组分成一定数量的batch，不同的是还增加了一个将batch随机化的的内容，所以生成这一步会在**REPET_rice_db**生成两个文件夹：
```
|-- batches  #分割后的batch
|-- batches_rnd  #随机化后的batch
|--other files
``` 

##### Step2 Align the reference TE sequences on each chunk  
这一步是使用Blaster、RepeatMasker、CENSOR这3种软件将chunk比对到参考的TE library（TEdenovo生成的结果，也可以使用其他的TE library）。由于CENSOR安装不成功，所以只用到了Blaster和RepeatMasker。除了正常的chunk比对到TE library，随机化的chunk也要和TE library做比对。所以这步会生成两个文件夹：
```sh
REPET_rice_TEdetect
REPET_rice_TEdetect_rnd
```
在两个文件夹内，每个方法都会生成相应的文件夹，里面包含自己的比对结果  

##### Step 3 Filter and combine HSP 
这一步是从step2的结果中过滤并合并HSPs, 然后**REPET_rice_TEdetect**目录下生成一个**Comb**的目录，里面包含每个batch比对的**newScores**文件和**newScores.clean_match.map**  

##### Step 4: Search for SSR & Step 5 Merge SSR annotations
使用TRF、REPEATMASKER、Mreps这3个软件寻找基因组中的SSR, 然后将这3个软件的结果整合，并将数据存入mysql数据库
运行结束会在生成**REPET_rice_SSRdetect**文件夹 
```sh
.
|-- Comb
|-- Mreps
|-- RMSSR
`-- TRF
```  
各目录内包含各软件的结果，comb目录内不含实际数据，整合后的数据在数据库中  

##### Step 6 Comparison with data banks 
这一步是将各batch比对到已有的重复序列数据库，水稻数据测试使用的是repbase的数据库，需要使用REPET专用的版本。分别比对到核酸数据和氨基酸数据，运行后会在**REPET_rice_TEdetect**目录下生成2个文件夹：
```
bankBLRtx
bankBLRx
```

##### Step 7 Remove spurious HSPs and long join procedure 
这一步会去除假的HSPs（比如全部由SSR组成的HSPs）并加入由片段链接起来的TEs
这步不会生成任何文件，但是在mysql里会生成相应的表，数据都存在数据库中  


##### Step 8 TE annotation export  
从最终的mysql数据库中的表格里导出TE的注释，可以选择生成GFF3或者gameXML的格式，每种格式都会生成对应的文件夹  
```sh
REPET_rice_gameXMLchr
REPET_rice_GFF3chr
```
以REPET_rice_GFF3chr为例：
```sh
|-- annotation_tables.txt
|-- chr01.gff3
|-- chr02.gff3
|-- chr03.gff3
|-- chr04.gff3
|-- chr05.gff3
|-- chr06.gff3
|-- chr07.gff3
|-- chr08.gff3
|-- chr09.gff3
|-- chr10.gff3
|-- chr11.gff3
|-- chr12.gff3
`-- chrnew01.gff3
```  
每条染色体都对应一个文件  

##### TIPS
1. TEdenovo和TEannot中均有寻找SSR的步骤：  
在TEdenovo的step5中，是在step4生成的重复序列的一致性序列中寻找SSR，目的是为了区分真正的TEs和SSRs,并在step7里将SSR过滤掉  
在TEannot中，是在整个基因组当中寻找SSR，目的是为了过滤掉某些看起来像是TE annotation但实际是完全由SSR覆盖的序列   

2. chunk和batch的关系
在step1中，会将基因组分成一定数量的chunk，每个chunk大小为200k, 每个chunk之间有10k的overlap。  
batch是一定数量的chunk的集合，这个数量的定义是：  
**minimum number of chunks per batch launched in parallel**  
即每个batch运行时，并行的最小chunk数。  
REPET中实际使用的其实是chunk，所有的操作都是基于chunk。batch只是代表了并行运行的chunk的数量

#### NOTE

##### note1：
REPET会自动把任务投递到集群上运行，在TEdenovo.cfg和TEannot.cfg中均可以修改相应的集群资源参数，包括使用的节点和要求的内存、CPU。ready.py那个脚本已经根据基因组大小对要求的资源做了一个粗略的区分。实际运行时如果觉得不妥当也可以人工去修改。  
![resource](http://oht4p9vad.bkt.clouddn.com/QQ%E5%9B%BE%E7%89%8720161219110044.png)  


##### note2
TEdenovo和TEannot各自均分为8小步，每一步运行完毕都会在输出日志文件中显示成功运行，如：
![success](http://oht4p9vad.bkt.clouddn.com/QQ%E6%88%AA%E5%9B%BE20161219110804.png)  
如果中间运行出错，也可以寻找相应步骤的日志文件，查看错误原因，然后重新运行。可以根据实际情况选择重新运行单步，或者全部重新运行。在TEdenovo和TEannot文件夹下，还有**run_TExxxxx.sh**,可以从这个文件运行。  

<u>重新运行前，请一定要运行下TEdenovo或TEannot目录下的**clean_jobs.sh**这个脚本，清除掉数据库中jobs表里的数据，不然之前的错误任务是不会停止的。</u>

##### note3
REPET投递的任务有时候会莫名的停止，特别是在做比对的时候。一般分为两种情况： 
1. 第一种是由于节点资源的原因，任务跑失败了，REPET会进行重投，默认重投是2次，2次均失败后，任务就会停止。所以修改了REPET源码里的重投次数，改成了6次。应该能避免这一类问题。
2. 第二种就比较诡异了，REPET调用的程序（如blaster)其实已经运行结束了，结果文件也产生了。但是数据库的jobs表中status字段依然还是"running"。而且通过qsee命令查看这个任务，也是运行的状态，但是cpu时间已经不动了。导致程序没有继续跑下去。所以这个时候需要去修改mysql表里的状态，可以使用TEdenovo或TEannot目录下的**change_status.sh**脚本：
```shell
sh ./change_status.sh xxxxxxx(通过qsee等命令看到的job_ID)
```
之后程序就能继续运行。这个问题目前都是随机出现，不知道原因出在哪里。


##### note4
如果遇上Grouper(TEdenovo-step3)报下面这类错误：  
```
grouper: /usr/lib64/libstdc++.so.6: version `GLIBCXX_3.4.15' not found (required by grouper)
```
这个错误是集群环境的原因，所以直接在环境变量中加入动态库就可以了。具体就在自己的.zshrc或.bashrc里加入
```
GCC_LIB="/nfs2/config/gcc/gcc-4.9.2/lib/:/nfs2/config/gcc/gcc-4.9.2/lib64/"
export PATH="$GCC_LIB:$PATH"  
```  

##### note5
Piler（TEdenovo-step3）由于自身软件原因，当基因组大小超过500M的时候，就会报“out of memory”的错误。  
此时可以直接注释掉运行Piler的步骤。  
同时将后续的步骤中的**-GrpRecPil**参数修改为**-GrpRec**。  


#### Advance  
由于REPET最后的输出结果只有GFF3文件或者gameXML文件，缺少统计信息。所以写了个脚本对输出的GFF3文件进行统计，输出结果样例如下：  
![statistic](http://oht4p9vad.bkt.clouddn.com/QQ%E6%88%AA%E5%9B%BE20161219113907.png)  

为了方便后续的基因注释等流程，还增加了一步使用repeatmasker对基因组文件中的重复序列进行注释的步骤（step4）。最后会在项目目录下输出xxxx.masked.fa文件。


### 资料
REPET官网：https://urgi.versailles.inra.fr/Tools/REPET  
TEdenovo文档： https://urgi.versailles.inra.fr/Tools/REPET/TEdenovo-tuto  
TEannot文档： https://urgi.versailles.inra.fr/Tools/REPET/TEannot-tuto  
demo服务器路径：/p299/user/og06/chenxiangjian1609/test_REPET/test_1220

REPET的文档十分详细，包括每一小步是做什么的，以及每一步运行的命令均有说明。  
ready.py生成的配置文件基本是按照默认参数生成的，实际运行中可以参考文档，对生成的配置文件进行修改。