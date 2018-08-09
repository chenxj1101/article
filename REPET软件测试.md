### TEdenovo
- Step 1 : Genomic sequences are cut into batches(将基因组按批次分割）
- Step 2 : The genome is aligned to itself using Blast（使用Blast将分割后的基因组进行自我比对）
- Step 2 structural : LTRs retrotransposons are searched in each batch using LTRharves（使用LTRharves从每一个批次中寻找LTR转座子）
- Step 3 : The repetitives HSP from BLAST are clustered by Recon, Grouper and/or Piler（使用Blast从Recon,Grouper和Piler的聚类结果中寻找重复的HSP）
- Step 3 structural : The predictions from LTRharvest are clustered using Blastclust or MCL（使用Blastclust或MCL聚合LTRharvest的预测结果）
- Step 4 : A multiple alignment is computed for each cluster, and a consensus sequence is derived from each multiple alignment（对每个聚类结果进行一次多序列比对，并得到一条最优的序列）
- Step 5 : Particular features are detected on each consensus, such as structural features or homology with known TE, HMM profiles or host genes（从每个最优结果中找到特有的特征，比如结构特征，已知TE，HMM profiles同源或者宿主基因）
- Step 6 : The consensus are classified using Wicker's classification（使用wicker分类法对consensus进行分类）
- Step 7 : SSR and under-represented unclassified consensus are filtered（对SSR，低代表性未分类的consensus进行过滤）
- Step 8 : The consensus are clustered into families to facilitate manual curation using Blastclust or MCL（将consensus聚合到家族中，并促进Blastclust或MCL的人工管理）


#### 要求  
数据格式要求为标准fasta格式，">"符号后面跟着的字符串中不能有空格，"=", ";", ":", "|"等符号  
序列要求每行为60个或者更少，序列名称要与项目名称相同  
需要有mysql数据库账号，许多数据包括任务投递命令均存储在Mysql数据库中


### TEannot  
- Step 1 genomic sequence and data banks preparation（准备基因组序列和数据库准备）
- Step 2 Align the reference TE sequences on each chunk （将每块的序列比对到参考TE序列）
- Step 3 Filter and combine HSP （过滤和合并 HSP）
- Step 4 Search for SSR （寻找SSR）
- Step 5 Merge SSR annotations  （合并SSR注释结果）
- Step 6 Comparison with data banks （比对到数据库）
- Step 7 Remove spurious HSPs and long join procedure （移除假的HSPs和长连接过程）
- Step 8 TE annotation export （TEannot输出）  


#### 要求  
输入数据：基因组fasta文件，TEdenovo输出的/*DmelChr4*_TEdenovo/*DmelChr4*_Blaster_GrpRecPil_Struct_Map_TEclassif_Filtered_Clustered/*DmelChr4*_denovoLibTEs_filtered_clustered.fa 作为**TE library**  
Step6所需要的核酸和氨基酸的数据库，需要fasta格式的文件，并直接放在TEannot的项目目录下，否则无法识别  
其他要求同TEdenovo  

#### 依赖软件  
已安装软件见
> setENV.sh 

```
export REPET_HOST=
export REPET_USER=
export REPET_PW=
export REPET_DB=
export REPET_PORT=

export REPET_PATH=/p299/user/og06/yukaicheng/biosoft/REPET_linux-x64-2.5
export PYTHONPATH=/lustre/project/og04/yukaicheng/Bin/new2.7.10/lib/python2.7/site-packages:/lustre/project/og04/yukaicheng/Bin/system_python_2.6/lib/python2.6/site-packages:$REPET_PATH
export REPET_JOBS=MySQL
export REPET_JOB_MANAGER=SGE
export REPET_QUEUE=SGE
#soft
export LTRHARVEST=/p299/user/og06/yukaicheng/biosoft/genometools-1.5.9/bin
export RECON=/p299/user/og06/yukaicheng/biosoft/RECON-1.08/scripts
export PILER=/nfs/pipe/genomics/biosoft/piler-pals/piler
export TRF=/nfs/pipe/genomics/biosoft/TRF
export HMMER=/p299/user/og06/chenxiangjian1609/software/hmmer/bin  #modify by chenxiangjian
export SHUFFLE=/p299/user/og06/chenxiangjian1609/software/hmmer-2.3.2/squid #modify by chenxiangjian
export RM=/p299/user/og06/chenxiangjian1609/software/RepeatMasker #modify by chenxiangjian
export MREPS=/p299/user/og06/chenxiangjian1609/software/mreps #modify by chenxiangjian
export MCL=/p299/user/og06/chenxiangjian1609/software/local/bin #modify by chenxiangjian
#
export PATH=$REPET_PATH/bin:$LTRHARVEST:$RECON:$PILER:$TRF:$HMMER:$SHUFFLE:$RM:$MREPS:$MCL:$PATH  #modify by chenxiangjian 
export LD_LIBRARY_PATH=/p299/user/og06/chenxiangjian1609/cpplib/lib:$LD_LIBRARY_PATH #modify by chenxiangjian
```

未安装软件如下：  
1. REPEATMASKER Sequence Search Engine：   
1.1 Cross_Match:  商业用途收费，研究或教育用途需要提供资料发送邮件获取  
1.2 WUblast: 以被ABBlast代替，收费。可以使用ncbi-blast代替
2. CENSOR：需要先安装bioperl模块  

### 运行  
具体文件见：  
>/p299/user/og06/chenxiangjian1609/test_REPET/TEannot2  
/p299/user/og06/chenxiangjian1609/test_REPET/TEdenovo2

#### TEdenovo  
文件准备：  
> test.fa  原始基因组文件，fasta格式  
format.py  自己写的对原始fa文件进行处理的脚本，使得格式能够满足REPET的要求  
setENV.sh  设置环境变量的脚本   
TEdenovo.cfg  TEdenovo运行时的配置文件，主要要修改的是项目名称和项目路径，第一次运行还需要修改相应的Mysql账号信息  
test_step.sh  TEdenovo运行脚本，修改里面的项目名称与配置文件里一致   
clean_jobs.sh  当流程投递的任务报错，而mysql中table : jobs中仍残存有该部分的投递任务信息时，用于清除 job信息  

先对原始文件进行处理：  
```
python  format.py  test.fa   denovo2.fa (输出文件的名称要与项目名称相同） 
```  

运行test_step.sh：  
```
nohup sh test_step.sh > denovo2.out 2>&1 &
```
也可以通过注释掉test_step.sh中的相应步骤，分步运行  

#### TEannot  
文件准备：
```
|-- annot2.fa (基因组文件）
|-- annot2_refTEs.fa -> /p299/user/og06/chenxiangjian1609/test_REPET/TEdenovo2/denovo2_Blaster_GrpRecPil_Struct_Map_TEclassif_Filtered_Blastclust/denovo2_denovoLibTEs_filtered_Blastclust.fa (TEdenovo最后步输出的结果作为TEs library)
|-- clean_jobs.sh  (当流程投递的任务报错，而mysql中table : jobs中仍残存有该部分的投递任务信息时，用于清除 job信息)
|-- repbase20.05_aaSeq_cleaned_TE.fa (REPET版的repbase氨基酸序列)
|-- repbase20.05_ntSeq_cleaned_TE.fa (REPET版的repbase核酸序列)
|-- setEnv.sh (设置环境变量的脚本)
|-- TEannot.cfg (TEannot运行的配置文件，与TEdenovo配置文件类似)
`-- test_annot_step.sh (TEannot运行脚本，与TEdenovo类似)
```  

运行test_annot_step.sh:  
```
nohup sh test_annot_step.sh > annot2.out 2>&1 &
```

