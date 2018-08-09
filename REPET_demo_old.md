首先放上demo的路径：  
> /p299/user/og06/chenxiangjian1609/test_REPET/REPET_demo

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
### STEP2、运行准备脚本  
ready.py脚本会对输入文件进行处理，使其格式满足REPET的要求，并重名成与项目名称一致（REPET要求），然后生成运行脚本以及需要的配置文件  
命令如下：  
```
python /p299/user/og06/chenxiangjian1609/software/REPET/ready.py -i demo.fa -p REPET_demo -g 55
```
参数解释：
- -i：输入文件  
- -p：项目名称，输入文件以此作为重命名名称
- -g: 基因组大小，单位为MB，可以通过"ll file"命令查看 

也可以通过-h参数获取帮助：
```
python /p299/user/og06/chenxiangjian1609/software/REPET/ready.py -h
```
会打印帮助信息，里面会给一个例子：
>-i: the input file, must  fasta file  
-p: the project name  
-g: the size of genome, the unit is MB  
-h: print this help  
Example:  
python ready.py -i test.fa -p project1 -g 55 

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

---------Please modify the  'TE***.cfg' and clean_jobs.sh use your mysql  account
---------Run the 'run***.sh' script in their directory
---------you can run script like 'nohup sh ./run***.sh > run***.out 2>&1 &'

```
**因为REPET的投递任务命令和数据是保存在数据库里的，所以要注意以下几点**  
**1.不同的人不要使用相同的数据库账号，以免同时运行项目时数据表里相互覆盖数据**  
**2.同一个人，项目名称不要跟以前取过的重复，否则会提示有些表已经存在，无法创建**  
**3.ready.py脚本运行完后，要手动修改2个配置文件和clean_jobs.sh中的数据库账号信息**   
**Mysql数据库的账号需要向cody哥申请**


生成的clean_jobs.sh（2个文件夹内均有此文件）如下： 
```
mysql -u your_username --port=3306 -h 10.1.1.216 -pyour_password_no_space_with'-p'<<EOF
use your_datebase_name;
delete from jobs;
EOF
exit;
```
修改后如下，以我自己的账号为例： 
```
mysql -u cxj_REPET --port=3306 -h 10.1.1.216 -pcxj_REPET20161101<<EOF
use cxj_REPET;
delete from jobs;
EOF
exit;
```
**注意：密码与'-p'和'<<EOF'之间没有空格** 

生成的配置文件TEdenovo.cfg和TEannot.cfg中，需要修改的部分如下：
```
[repet_env]
repet_version: 2.5
repet_host: #mysql host
repet_user: #your mysql username
repet_pw: #your mysql password
repet_db: #your database name
repet_port: 3306
repet_job_manager: SGE
```  

修改后如下：  
```
[repet_env]
repet_version: 2.5
repet_host: 10.1.1.216
repet_user: cxj_REPET
repet_pw: cxj_REPET20161101
repet_db: cxj_REPET
repet_port: 3306
repet_job_manager: SGE
```  
修改完毕后，就可以进入TEdenovo文件夹，运行脚本  


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


#### Other  
REPET官网：https://urgi.versailles.inra.fr/Tools/REPET/  

关于 ready.py：目前因为不管输入文件的格式是否符合要求，都会调用itools对其进行格式化和重命名，所以会花费一点时间，基因组越大时间越长。如遇到问题或疑问，可以直接联系我：chenxiangjian1609@1gene.com.cn