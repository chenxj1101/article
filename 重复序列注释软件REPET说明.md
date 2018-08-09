### 简介
REPET是一个寻找基因组中的重复序列并进行注释和分析的软件。主要分为两个部分  
1. TEdenovo
2. TEannot

**TEdenovo**部分是通过denovo的方式找寻重复序列，最后得到一个TE sequences的库。  

![TEdenovo](https://urgi.versailles.inra.fr/var/storage/images/media/images/te_denovo/8252-4-eng-GB/TE_DeNovo.png)  

**TEannot**部分是konowledge based的方式找寻重复序列，使用的库既可以是TEdenovo生成的库，也可以是其他的库。最后将注释以GFF3或gameXML的形式输出。  

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
REPET要求基因组文件为标准的fa文件，即每行的碱基数为60个，所以脚本里调用了Itools进行格式化。  
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
##### note1：
REPET会自动把任务投递到集群上运行，在TEdenovo.cfg和TEannot.cfg中均可以修改相应的集群资源参数，包括使用的节点和要求的内存、CPU。ready.py那个脚本已经根据基因组大小对要求的资源做了一个粗略的区分。实际运行时如果觉得不妥当也可以人工去修改。  
![resource](http://oht4p9vad.bkt.clouddn.com/QQ%E5%9B%BE%E7%89%8720161219110044.png)  


##### note2
TEdenovo和TEannot各自均分为8小步，每一步运行完毕都会在输出日志文件中显示成功运行，如：
![success](https://oht4p9vad.bkt.clouddn.com/QQ%E6%88%AA%E5%9B%BE20161219110804.png)  
如果中间运行出错，也可以寻找相应步骤的日志文件，查看错误原因，然后重新运行。可以根据实际情况选择重新运行单步，或者全部重新运行。在TEdenovo和TEannot文件夹下，还有**run_TExxxxx.sh**,可以从这个文件运行。  

<u>重新运行前，请一定要运行下TEdenovo或TEannot目录下的**clean_jobs.sh**这个脚本，清除掉数据库中jobs表里的数据，不然之前的错误任务是不会停止的。</u>  

##### note3
REPET投递的任务有时候会莫名的停止，特别是在做比对的时候。一般分为两种情况： 
1. 第一种是由于节点资源的原因，任务跑失败了，REPET会进行重投，默认重投是2次，2次均失败后，任务就会停止。所以修改了REPET源码里的重投次数，改成了6次。应该能避免这一类问题。
2. 第二种就比较诡异了，REPET调用的程序（如blaster)其实已经运行结束了，结果文件也产生了。但是数据库的jobs表中status字段依然还是"running"。而且通过qsee命令查看这个任务，也是运行的状态，但是cpu时间已经不动了。导致程序没有继续跑下去。所以这个时候需要去修改mysql表里的状态，可以使用TEdenovo或TEannot目录下的change_status.sh脚本：
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




##### other  
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