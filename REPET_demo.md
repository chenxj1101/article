试试markdown
首先放上demo的路径：  
> /p299/user/og06/chenxiangjian1609/test_REPET/test3/  

REPET官网：https://urgi.versailles.inra.fr/Tools/REPET/ 

### STEP1、文件准备
REPET是一个对基因组进行注释的流程，包括TEdenovo和TEannot两个部分。这个流程运行需要用到一些配置文件，我写了个Python的脚本用来自动生成需要的配置文件和运行脚本等。  

脚本路径：  
>/p299/user/og06/chenxiangjian1609/software/REPET/ready.py  

建议为REPET单独建一个文件夹，如：  
```
mkdir  REPET_demoo
```  
然后进入文件夹，将基因组fasta文件复制或链接到该目录：  
```
cd REPET_demo  
cp your/genome/fasta/file/path.fa  ./
```  
新建数据库配置文件，里面写入数据库配置  
```
touch db.cfb  #新建文件  
```  
打开，写入数据库配置：
```
cxj_REPET    #数据库账号
cxj_REPET20161101    #数据库密码
cxj_REPET    #自己使用的数据库的名称
```

### STEP2、运行准备脚本  
ready.py脚本会对输入文件进行处理，使其格式满足REPET的要求，并重命名成与项目名称一致（REPET要求），然后生成运行脚本以及需要的配置文件  
命令如下：  
```
python /p299/user/og06/chenxiangjian1609/software/REPET/ready.py -i demo.fa -p REPET_demo -g 55 -d db.cfg
```
参数解释：
- -i：输入文件  
- -p：项目名称，输入文件以此作为重命名名称
- -g: 基因组大小，单位为MB，可以通过"ll file"命令查看 
- -d: 数据库配置文件，里面包含数据库账户信息
也可以通过-h参数获取帮助：
```
python /p299/user/og06/chenxiangjian1609/software/REPET/ready.py -h  
```  
输出如下：  
```
[$] <> python /p299/user/og06/chenxiangjian1609/software/REPET/ready.py -h                                      

           -i: the input file, must  fasta file
           -p: the project name
           -g: the size of genome, the unit is MB
           -d: the database config file
           -h: print this help
Example:
           python ready.py -i test.fa -p project -g 55 -d db.cfg
```

ready.py脚本运行后会在屏幕上输出当前目录的目录树，并提示手工修改配置文件中数据库相关信息：  
```
.
|-- demo.fa
|-- REPET_demo.fa
|-- TEannot
|   |-- clean_jobs.sh  （当任务运行出错中断，jobs表中的数据没有清空时，清空jobs表的脚本.#非打印内容）
|   |-- repbase20.05_aaSeq_cleaned_TE.fa
|   |-- repbase20.05_ntSeq_cleaned_TE.fa
|   |-- REPET_demo.fa -> /p299/user/og06/chenxiangjian1609/test_REPET/REPET_demo/REPET_demo.fa
|   |-- run_TEannot.sh
|   |-- setEnv.sh
|   `-- TEannot.cfg
`-- TEdenovo
    |-- clean_jobs.sh
    |-- REPET_demo.fa -> /p299/user/og06/chenxiangjian1609/test_REPET/REPET_demo/REPET_demo.fa
    |-- run_TEdenovo.sh
    |-- setEnv.sh
    `-- TEdenovo.cfg

2 directories, 14 files

---------Run the 'run***.sh' script in their directory
---------you can run script like 'nohup sh ./run***.sh > run***.out 2>&1 &'

```
**因为REPET的投递任务命令和数据是保存在数据库里的，所以要注意以下几点**  
**1.不同的人不要使用相同的数据库账号，以免同时运行项目时数据表里相互覆盖数据**  
**2.同一个人，项目名称不要跟以前取过的重复，否则会提示有些表已经存在，无法创建**  
**3.Mysql数据库的账号需要向cody哥申请**



### STEP3、运行REPET  
首先进入TEdenovo目录，运行run_TEdenovo.sh脚本：
```
cd TEdenovo
nohup sh ./run_TEdenovo.sh > run_TEdenovo.out 2>&1 &
```  
考虑运行时间的关系，推荐使用"nohuo XXXXX &"将任务放在后台运行  
nohup提交任务后，可以用jobs命令查看任务状态（仅当前登录有效）  
脚本里的每一步都会产生相应的stepX.o日志文件，里面可以查看任务运行的状态，如果里面有   
"step X finished successfully" (X为数字)  
则表示这一步运行成功。

run_TEdenovo运行完毕后，进入TEannot目录，运行run_TEannot.sh脚本：
```
cd ../TEannot
nohup sh ./run_TEannot.sh > run_TEannot.out 2>&1 &
```
run_TEannot.sh脚本运行完毕后，代表REPET运行结束 


如果遇到任务运行失败，首先运行clean_jobs.sh脚本，清除数据库中的jobs表数据，然后根据实际情况处理，如删除已经产生的文件，重新运行，或者注释掉已经运行的部分，继续运行等。  


#### 问题：   
**Q1**：水稻基因组测试的时候，TEannot第二步跑blaster部分，按照默认参数，会有几个Batch跑不下去  
**A1**: 默认参数，sensitivity这个参数的值是2，按这份数值运行，blaster-9会停在query88，blaster-24会停在query108，blaster-43会停在query28.一开始以为是节点的原因，结束命令并修改配置文件，将之前运行的节点去除，并将sensitivity的值改为4(可改1-4).开始运行后，还是会停住，而且停住的batch更多了，分别为-3，-7，-19，-37，-43，-54， -59，-61.在数值为2时停住的-9和-24则跑完了，-43依然停在query28的位置。再将sensitivity的值改为1，重新运行，这次全部运行通过。怀疑可能是基因组的原因，如果是基因组的原因，REPET应该需要更多的基因组来测试。REPET加入到组装流程还需要评估，如果某些基因组在sensitivity参数为1时，还是会停住，怎么解决？    

**Q2**：水稻基因组在跑step2_r的repeatmasker和step3的matcher的时候，还是会停住，程序已经运行完毕了，但是数据库的任务状态没有更新，导致无法继续运行  
**A2**：通过修改mysql中的任务状态，可以使得任务重新运行，如  
>update jobs set status="error" where jobid=xxxxxxx  

repeatmasker修改任务状态重新运行后，停止的任务都跑完了。  

step3的matcher修改状态重新运行，还是有16个子任务停在那里了  

**Q3**：TEannot中step3的blaster的运行时间和什么有关？  
**A3**：跑了2次step3，通过对比两次的结果，发现运行时间长的batch基本是重合的，如图所示：   
            ![image](http://upload-images.jianshu.io/upload_images/23987-bd9966e8fcf8b0ac.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)  
          
          
**Q4**：TEdenovo的step2中，自身比对使用blaster所需的最大内存和什么有关？  
**A4**：综合真菌，水稻，大额牛的情况来看，最大内存为基因组大小的10倍左右，是否跟线程参数-n有关系？现在-n 的值为10，默认为1。分别测试了水稻基因组（364M）在-n参数为10和1的情况的最大内存，10时的最大内存为2.27G，1的时候为1.73G。可见使用的CPU数量多了，内存的使用量也会相应增加。  
