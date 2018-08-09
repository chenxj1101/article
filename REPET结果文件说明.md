  
水稻REPET路径  
>/p299/user/og06/chenxiangjian1609/test_REPET/REPET_rice364 
### 水稻REPET结果文件说明  
使用水稻基因组数据对REPET软件进行了测试，目前软件已经可以跑通，现在对各步骤产生的文件做一个说明，也能更好的帮助理解REPET的使用。  

#### Part1 TEdenovo  
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
第二步是将step1中的每个batch与由chunk合成的基因组进行自身比对寻找HSPs（high-scoring segment pairs），使用的是REPET对自己封装blast。这一步花费的时间较长，对于大基因组来说占用的内存也较大。  
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
.
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

#### Part2  TEannot  

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
这一步会去除假的HSPs（比如全部由SSR组成的HSPs）并加入因为其他原因被忽略的HSPs
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