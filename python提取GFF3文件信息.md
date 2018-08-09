#### 前言
因为最近使用REPET这个程序对基因组进行重复序列的注释，但是最后输出的结果是GFF3格式的文件，缺少统计信息。因此用python写了个脚本，对GFF3的信息进行提取并统计。

#### GFF3文件
想要从GFF3文件中提取的信息为重复序列的种类，数量，以及bp数。  
GFF3文件分为9列，这次用到是第2、4、5、9列，分别代表来源，序列起始位置，结束位置，以及属性。  
重复序列的种类可以从第9列属性中提取，bp数以（结束位置-起始位置+1）进行计算，数量则在过程中统计。

#### 思路
按行读入GFF3文件，以"**\t**"符号分解。首先通过第2列，判断来源，有3种来源：  
1. TEs
2. SSRs
3. blastx or tblasts

不同的来源，后缀不同，通过判断相应的字符串是否在第2列字符串中来判断是哪种来源。再通过第9列属性字符串，判断相应的分类代码字符串是否在其中，如果在其中，则将分类代码放入一个列表，然后以分类代码为键，bp数为值生成字典。  
for循环不断读入，列表增长，字典合并，最后将列表唯一化，通过Counter类生成相应的字典，这个字典的值就是各分类的数量。  

#### 代码
生成字典是通过事先定义一个函数的方式实现的：  
```python
def return_TEs(lines):
    TEs=['RYX', 'RXX-LARD', 'RIX', 'RLX', 'RPX', 'RSX', 'RXX-TRIM', 'DYX', 'DHX',
    'DXX-MITE', 'DMX', 'DTX', 'RXX-chim', 'DXX-chim', 'noCat']
    (seqid, source, seqtype, start, end, score, strand, phase, attributes) =\
	lines.split('\t')
    te_len = int(end) - int(start) + 1
    for te in TEs:
        if te in attributes:
            return {te:te_len}

```

字典合并的函数：  
```python
#字典合并的函数，即相同的键的值进行相加
def union_dict(*objs):
    _keys = set(sum([obj.keys() for obj in objs],[]))
    _total = {}
    for _key in _keys:
        _total[_key] = sum([obj.get(_key,0) for obj in objs])
    return _total

```

附上完整代码：  
```python
#!/usr/bin/env python
#-*- coding: utf-8 -*-

"""
根据REPET输出的GFF3文件统计重复序列的信息
"""

import os
from collections import Counter
#使用字典返回TEdenovo的结果
def return_TEs(lines):
    TEs=['RYX', 'RXX-LARD', 'RIX', 'RLX', 'RPX', 'RSX', 'RXX-TRIM', 'DYX', 'DHX',
    'DXX-MITE', 'DMX', 'DTX', 'RXX-chim', 'DXX-chim', 'noCat']
    (seqid, source, seqtype, start, end, score, strand, phase, attributes) =\
	lines.split('\t')
    te_len = int(end) - int(start) + 1
    for te in TEs:
        if te in attributes:
            return {te:te_len}
#使用字典返回SSR的结果
def return_SSRs(lines):
    (seqid, source, seqtype, start, end, score, strand, phase, attributes) =\
	lines.split('\t')
    ssr_len = int(end) - int(start) + 1
    return ['SSR', ssr_len]
#使用字典返回blastx和tblastx的结果
def return_Blast(lines):
    blsat_te={'TRIM':'RXX-TRIM', 'LTR':'RLX', 'MITE':'DXX-MITE', 'Crypton':'DYX',
    'TIR':'DTX', 'Helitron':'DHX', 'PLE':'RPX', 'DIRS':'RYX', 'LINE':'RIX',
    'SINE':'RSX', 'Maverick':'DMX', 'ClassI:?':'RXX-chim', 'ClassII:?':'DXX-chim'} #通过字典将REPET中的分类代码与RepBase的分类代码对应起来
    (seqid, source, seqtype, start, end, score, strand, phase, attributes) =\
	lines.split('\t')
    blast_len = int(end) - int(start) + 1
    for te in blsat_te:
        if te in attributes:
            return {blsat_te[te]:blast_len}
#字典合并的函数，即相同的键的值进行相加
def union_dict(*objs):
    _keys = set(sum([obj.keys() for obj in objs],[]))
    _total = {}
    for _key in _keys:
        _total[_key] = sum([obj.get(_key,0) for obj in objs])
    return _total

os.system('cat *.gff3 > genome.gff3') #合并REPET生成的所有gff3文件
te = 'REPET_TEs'
ssr = 'REPET_SSRs'
blast = 'REPET_blastx'
tblast = 'REPET_tblastx'
genome_size=0
ssr_num, ssr_len=[0]*2
TEs=[]
TEs_dict={}
Rep=[]
Rep_dict={}
with open('genome.gff3', 'r') as gff: #genome.gff3即前面合并后生成的文件
    for line in gff:
        if line.startswith('##g'):
            pass
        elif line.startswith('##s'):
            (t1, t2, t3, length) = line.split(' ')
            genome_size += int(length)  #计算基因组总长度
        else:
            if te in line:
                a = return_TEs(line)
                TEs = TEs + a.keys()  #种类存入列表
                TEs_dict = union_dict(TEs_dict, a) #将每一行的结果放入字典
            if ssr in line:
                b = return_SSRs(line)
                ssr_num += 1  #计算SSR的个数
                ssr_len += b[1]  #计算SSR的总长度

            if blast in line or tblast in line:
                c = return_Blast(line)
                Rep = Rep + c.keys()
                Rep_dict = union_dict(Rep_dict, c)

zero={'RYX':0, 'RXX-LARD':0, 'RIX':0, 'RLX':0, 'RPX':0, 'RSX':0, 'RXX-TRIM':0,
 'DYX':0, 'DHX':0,'DXX-MITE':0, 'DMX':0, 'DTX':0, 'noCat':0, 'RXX-chim':0, 'DXX-chim':0}  #创建一个包含所有类型代码的字典，然后将之前的结果与其相加，避免空值
TEs_type = union_dict(dict(Counter(TEs)), zero) # 列表里的重复值唯一化，并计算重复的次数（相当于每种类型的个数)
Rep_type = union_dict(dict(Counter(Rep)), zero)
TEs_dict = union_dict(TEs_dict, zero)
Rep_dict = union_dict(Rep_dict, zero)
all_type = union_dict(TEs_type, Rep_type, zero)
all_len = union_dict(TEs_dict, Rep_dict, zero)

```