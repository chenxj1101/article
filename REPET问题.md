**LARD**：large retrotransposon derivative （大型反转座子衍生物？）  
**TRIM**：terminal-repeat retrotransposons in miniature （微型末端重复反转座子）  
**HSP**： The High-scoring Segment Pair。  
The High-scoring Segment Pair (HSP) is the fundamental unit 
     of BLAST algorithm output.  An HSP consists of two sequence
     fragments of arbitrary but equal length whose  alignment  is
     locally  maximal  and for which the alignment score meets or
     exceeds a threshold or cutoff score.  A set of HSPs is  thus
     defined  by  two  sequences,  a scoring system, and a cutoff
     score; this set may be empty if the cutoff score  is  suffi-
     ciently  high.   In  the programmatic implementations of the
     BLAST algorithm described here, each HSP consists of a  seg-
     ment  from  the  query  sequence  and  one  from  a database
     sequence.  The sensitivity and speed of the programs can  be
     adjusted  via  the standard BLAST algorithm parameters W, T,
     and X (Altschul et al., 1990); selectivity of  the  programs
     can be adjusted via the cutoff score.
     
**long join**：Because the input genomic sequences may contain large regions of heterochromatin, some TEs are expected to be nested. As a given copy can be interrupted by several other TEs inserted more recently, we expect to find distant fragments belonging to the same copies.  
**由于基因组中含有大量异染色质的区域，所以一些TEs会相互嵌套。一个拷贝可能会被多个TEs打算，我们希望找到来自同一个拷贝的距离较远的分离片段**  
MATCHER is used at step 3, not only to filter overlapping HSPs, but also to join them. However, it relies on a scoring scheme that, in some extreme cases (deep nesting, distant fragmentation), appears to be unsufficient. Therefore we implemented a "long join procedure" aimed at recovering the join of these fragments missed sometimes by MATCHER.  
**MATCHER在第3步不仅过滤重叠的HSP，还会加入HSP.这个方法依赖的是评分方案，对于某些极端情况还是不适用的**  

Fragments involved in nesting patterns must respect the three following constraints: (i) be co-linear; (ii) have the same age, and (iii) be separated by younger TE insertions. **The identity percentage with a reference consensus sequence is used to estimate the age of a copy**. Consecutive fragments on both the genome and the same reference TE were automatically joined if they respect these constraints. We call them "nest join".  
**片段如果被定义为嵌套的话必须遵循3个条件：(i)共线性；(ii)有相同的age；(iii)被更年轻的TE插入分离**。

Sometimes large non-TE sequence insertions can be observed in a TE copy. They are suspected to appear by gene conversion. In order to deal with these cases, we also join fragments if they are separated by an insert of less than 5kb and/or less than 500bp of mismatches, and have the same age. We call this a "simple join".  
**有时会在一个TE拷贝里发现大量的non-TE序列插入，怀疑是来自基因基因转换。为了处理这种情况，我们同样加入这些片段如果他们被少于5kb或500bp的不匹配片段分离，并有相同的age**

Young copies are expected to keep longer fragments than old copies, because deletions accumulate with time. This is a final control of nested patterns based on a different assumption than consensus nucleotide identity percentage (see above). Thus, at the end, nested TEs are split if inner TE fragments are longer than outer joined fragments. They are reported as "split".   
**年轻的拷贝猜想比老的拷贝保持了更长的片段，因为随着时间删除会不断积累。这是嵌套模式的最终控制基于和一致性核苷酸身份比例所不同的假设**


**SSR找了2次(TEdenovo, TEannot)都有：**  
In the TEdenovo pipeline (step 5), we detect SSRs in de novo consensus of repeated sequences previously built at step 4.  
The aim of this analysis is to discriminate between true TEs and SSRs.  
At the end of TEdenovo (step 7), we advise to filter them (see the TEdenovo tutorial for more details).  
In the TEannot pipeline (step 4), we detect SSRs in the genome sequences.  
The aim here is to filter out what seems to be TE annotations but is completely covered by SSRs.  

在TEdenovo的step5中，是在step4生成的重复序列的一致性序列中寻找SSR，目的是为了区分真正的TEs和SSRs,并在step7里将SSR过滤掉  
在TEannot中，是在整个基因组当中寻找SSR，目的是为了过滤掉某些看起来像是TE annotation但实际是完全由SSR覆盖的序列  

**chunk和batch的关系**  
在step1中，会将基因组分成一定数量的chunk，每个chunk大小为200k, 每个chunk之间有10k的overlap。  
batch是一定数量的chunk的集合，这个数量的定义是：  
**minimum number of chunks per batch launched in parallel**  
即每个batch运行时，并行运行的最小chunk数。  
REPET中实际使用的其实是chunk，所有的操作都是基于chunk。batch只是代表了并行运行的chunk的数量（=batch大小/200k）