#### fastqc_raw(1) 
- rules: qc_raw.rules
- input: raw/{prefix}.fastq.gz
- output: qc/raw/{prefix}_fastqc.html,  qc/raw/{prefix}_fastqc.zip
- shell: /nfs2/pipe/Re/Software/miniconda/bin/fastqc -t {threads} -j {java} {input} -o qc/raw



#### readfq_raw(2)
- rules: qc_raw.rules
- input: raw/{prefix}.fastq.gz
- output: qc/raw/{prefix}_fqstat.txt
- shell: /nfs2/biosoft/bin/itools Fqtools stat -CPU {threads} -InFq {input} -OutStat {output} || echo '$? not eqaul to 0'



#### panel_bed_stat(3)
- rules: qc_panel.rules
- input: /lustre/project/og04/pub/database/panel_ca_pm/panel.bed(有变化，原来是：/lustre/project/og04/pub/database/onco_panel/cancer_panel_annotation_nohap.bed）
- output: qc/panel/panel_bed_stat.txt
- shell:   

```
echo -e "#panel_bed\t$(readlink -e {input})" > {output}
paste <(echo "#region_count") <(grep -v '^#' {input} | wc -l) >> {output}
grep -v '^#' {input} \
    | \
awk ' \
    BEGIN {{ \
        suml = 0; \
    }} \
    {{ \
        len = $3 - $2; \
        suml += len; \
    }} \
    END {{ \
        print "#region_len" "\t" suml \
    }} \
' >> {output}
GENE_n=$(grep -v '^#' {input} \
    | cut -f 4 | cut -d ',' -f1 \
    | sort | uniq -c | wc -l)
echo -e "#gene_count\t$GENE_n" >> {output}
```

#### align_speedseq(4)
- rules: align_speedseq.rules
- input: lambda wildcards: config["raw"][wildcards.sample]
- output: align/{sample}.bam, align/{sample}.bam.bai, align/{sample}.discordants.bam, align/{sample}.discordants.bam.bai, align/{sample}.splitters.bam, align/{sample}.splitters.bam.bai
- shell: /lustre/project/og04/pub/biosoft/speedseq/bin/speedseq align -v -t {thread} -o align/{wildcards.sample} -R "@RG\\tID:{wildcards.sample}\\tSM:{wildcards.sample}\\tLB:{wildcards.sample}" {fa} {input}



#### filter_SOAPfilter(5)
- rules: filter_raw.rules
- input: lambda wildcards: config["raw"][wildcards.sample]
- output: clean/{sample}_R1.fastq.gz, clean/{sample}_R2.fastq.gz, clean/{sample}_filter.log, clean/{sample}_filterstat.txt
- shell: /lustre/project/og04/pub/script/filter_raw {input} -o clean -@ {thread} -s {output.stat} {params.prefix} {params.insert_size} {params.trim} {params.quality} {params.custom}



#### summary_fqstat_raw(6)
- rules: qc_raw.rules
- input:qc/raw/{sample}_R1_fqstat.txt", sample = config["sample"], qc/raw/{sample}_R2_fqstat.txt", sample = config["sample"]
- output: qc/raw/fqstat_summary.txt, qc/raw/fqstat_summary_pe.txt
- shell: /lustre/project/og04/pub/script/summary_fqstat qc/raw



#### fastqc_clean(7)
- rules: qc_clean.rules
- input: clean/{prefix}.fastq.gz
- output: qc/clean/{prefix}_fastqc.html, qc/clean/{prefix}_fastqc.zip
- shell: /nfs2/pipe/Re/Software/miniconda/bin/fastqc -t {threads} -j {java} {input} -o qc/clean


#### readfq_clean(8)
- rules: qc_clean.rules
- input: clean/{prefix}.fastq.gz
- output: qc/clean/{prefix}_fqstat.txt
- shell: /nfs2/biosoft/bin/itools Fqtools stat -CPU {threads} -InFq {input} -OutStat {output} || echo '$? not equal to 0'


#### multiqc_raw(9)
- rules: qc_raw.rules
- input: qc/raw/{sample}_R1_fastqc.html", sample = config["sample"], qc/raw/{sample}_R2_fastqc.html", sample = config["sample"]
- output: qc/raw/multiqc_report.html
- shell: /nfs2/biosoft/bin/multiqc -f .


#### samtools_stats_bam(10)
- rules: qc_align.rules
- input: lambda wildcards: config['align'][wildcards.sample]
- output: qc/align/{sample}.samtools_stat.txt
- shell: /nfs2/pipe/Re/Software/miniconda/bin/samtools stats {input} > {output}


#### qualimap(11)
- rules: qc_align.rules
- input: lambda wildcards: config['align'][wildcards.sample]
- output: qc/align/{sample}.qualimap/qualimapReport.html
- shell: sh /lustre/project/og04/pub/script/qualimap.sh {input} qc/align/{wildcards.sample}.qualimap


#### cnv_cnvkit_germline(12)
- rules: var_cnv.rules
- input: lambda wildcards: config['align'][wildcards.germline], ref_cnn=/lustre/project/og04/pub/database/panel_ca_pm/cnv_cnn/panel.cnn
- output: cnv/{germline}/{germline}.cns, cnv/{germline}/{germline}.cnr, cnv/{germline}/{germline}.gainloss.tsv, cnv/{germline}/{germline}.cnv.tsv
- shell: /lustre/project/og04/pub/script/cnv/cnvkit -m germline -2 '{input.bam_t}' -r {input.ref_cnn} -o cnv/{wildcards.germline} -p {wildcards.germline} -b {panel_bed} -f {fa} -t {thread}


#### samtools_depth_bed_bam(13)
- rules: qc_panel.rules
- input: lambda wildcards: config['align'][wildcards.sample]
- output: qc/panel/{sample}.samtools_depth_bed.txt
- shell: /nfs2/pipe/Re/Software/miniconda/bin/samtools depth -b {panel_bed} {input} > {output}


#### samtools_mpileup(14)
- rules: prepare_mpileup.rules
- input: align/{prefix}.bam
- output: align/{prefix}.mpileup
- shell: /nfs2/pipe/Re/Software/miniconda/bin/samtools mpileup -A -C 50 -d 99999 -E -L 99999 -q 0 -Q 0 -x -f {fa} -o {output} {input}


#### generate_coverage(15)
- rules: qc_covmap.rules
- input: align/{sample}.bam, /lustre/project/og04/pub/database/human_genome_hg19/noGL/human_g1k_v37_noGL.fasta.cov.index
- output: qc/covmap/{sample}.coverage
- shell: /nfs2/pipe/Re/Software/miniconda/bin/bedtools genomecov -ibam {input.bam} -g {input.index} > {output}


#### sv_germline_speedseq(16)
- rules: var_germline_speedseq
- input: lambda wildcards: config["align"][wildcards.germline]
- output: var_speedseq/{germline}.sv.vcf.gz
- shell: /lustre/project/og04/pub/biosoft/speedseq/bin/speedseq sv -v -x {speedseq_bed}/lumpy.exclude.bed -t {thread} {params.custom} -o var_speedseq/{wildcards.germline} -R {fa} -B {input} -S align/{wildcards.germline}.splitters.bam -D align/{wildcards.germline}.discordants.bam || echo '-> no sv'  # to avoid non-zero exit status due to none sv called


#### sv_factera(17)
- rules: var_sv.rules
- input: align/{prefix}.bam
- output: sv/{prefix}.fusion.factera.log
- shell: 
```
export PATH=/nfs/pipe/Re/Software/miniconda/bin:$PATH
[[ ! -d sv/{wildcards.prefix}_factera ]] && mkdir -pv sv/{wildcards.prefix}_factera
/lustre/project/og04/pub/biosoft/bin/factera -C -o sv/{wildcards.prefix}_factera {input} {panel_bed} {fa}.2bit || echo 'no fusion from factera'
cp -v 'sv/{wildcards.prefix}_factera/{wildcards.prefix}.factera.fusions.txt' 'sv/{wildcards.prefix}.fusion.factera.tsv' && echo 'done factera' > {output} || echo 'no fusion from factera' > {output}

```


#### itools_stat_bam(18)
- rules: qc_align.rules
- input: lambda wildcards: config['align'][wildcards.sample]
- output: qc/align/{sample}.itools_stat.txt, qc/align/{sample}.itools_depth.gz
- shell: 
```
/nfs2/biosoft/bin/itools Xamtools stat -InFile {input} -Bam -OutStat {output.stat} -Ref {fa} -SiteD 3 || echo 'itools stat done'
mv -v {output.stat}.depth.gz {output.gz}
```


#### samstat(19)
- rules: qc_align.rules
- input: lambda wildcards: config['align'][wildcards.sample]
- output: qc/align/{sample}.samstat.html
- shell: 
```
export PATH="/nfs2/pipe/Re/Software/miniconda/bin":$PATH
/nfs2/pipe/Re/Software/bin/samstat {input} && mv {input}.samstat.html {output}
```


#### bedtools_genomecov_bam(20)
- rules: qc_align.rules
- input: align/{sample}.bam, /lustre/project/og04/pub/database/human_genome_hg19/noGL/human_g1k_v37_noGL.fasta.cov.index
- output: qc/align/{sample}.bedtools_genomecov.txt
- shell: /nfs2/pipe/Re/Software/miniconda/bin/bedtools genomecov -bga -split -ibam {input.bam} -g {input.index} > {output}


#### qualimap_panel(21)
- rules: qc_align.rules
- input: lambda wildcards: config['align'][wildcards.sample]
- output: qc/align/{sample}.qualimap_panel/qualimapReport.html
- shell: 
```
[[ ! -f ref/panel.bed.6col ]] && awk 'OFS="\t"{{print $1, $2, $3, "TRANSCRIPT", "0", "+"}}' {panel_bed} > ref/panel.bed.6col && sleep 5
sh /lustre/project/og04/pub/script/qualimap.sh {input} qc/align/{wildcards.sample}.qualimap_panel ref/panel.bed.6col
```


#### calc_mapping_ratio(22)
- rules: qc_covmap.rules
- input: align/{sample}.bam
- output: qc/covmap/{sample}.map
- shell:
```
total=`{samtools} view -c {input}`
mapped=`{samtools} view -c -F 4 {input}`
#unmapped=`{samtools} view -c -f 4 {input}`
map_rate=`perl -e \"print (substr \$mapped/\$total, 0, 6)\"`
echo \"mapping\t$map_rate\" > {output}
```


#### fastqc_bam(23)
- rules: qc_align.rules
- input: lambda wildcards: config['align'][wildcards.sample]
- output: qc/align/{sample}.fastqc.html
- shell: /nfs2/pipe/Re/Software/miniconda/bin/fastqc -t {thread} -j {java} {input} -o qc/align && mv qc/align/{wildcards.sample}_fastqc.html {output}


#### samtools_depth_all_bam(24)
- rules: qc_align.rules
- input: lambda wildcards: config['align'][wildcards.sample]
- output: qc/align/{sample}.samtools_depth.txt.gz
- shell: /nfs2/pipe/Re/Software/miniconda/bin/samtools depth {input} | gzip > {output}


#### snp_germline_speedseq(25)
- rules: var_germline_speedseq.rules
- input: lambda wildcards: config["align"][wildcards.germline]
- output: var_speedseq/{germline}.var.vcf.gz
- shell: /lustre/project/og04/pub/biosoft/speedseq/bin/speedseq var -v -t {thread} -o var_speedseq/{wildcards.germline}.var {fa} {input}


#### plot_bamstats(26)
- rules: qc_align.rules
- input: rules.samtools_stats_bam.output (qc/align/{sample}.samtools_stat.txt)
- output: qc/align/{sample}.plot_bamstats/
- shell: /nfs2/pipe/Re/Software/miniconda/bin/plot-bamstats -p {output} {input}


#### samtools_stats_bam_dups(27)
- rules: qc_align.rules
- input: qc/align/{sample}.samtools_stat.txt
- output: qc/align/{sample}.samtools_dups.txt
- shell:
```
total_reads=$(grep '^SN\sraw total sequences' {input} | cut -f 3)
dup_reads=$(grep '^SN\sreads duplicated' {input} | cut -f 3)
echo $(perl -e "print substr ($dup_reads / $total_reads, 0, 6)") > {output}
```


#### boxplot_panel_depth_samtools(28)
- rules: qc_panel.rules
- input: qc/panel/{sample}.samtools_depth_bed.txt
- output: qc/panel/{sample}.boxplot_panel_depth_samtools.png
- shell: sh /lustre/project/og04/pub/pipeline/plot_depth_distribution/panel_depth_boxplot.sh -i {input} -p {panel_bed} -o qc/panel -n {wildcards.sample}.boxplot_panel_depth_samtools


#### prepare_vcf_sv(29)
- rules: prepare_vcf.rules
- input: var_speedseq/{prefix}.sv.vcf.gz
- output: var/{prefix}.sv.vcf, var/{prefix}.sv.vcf.gz
- shell: 
```
if [[ $(zgrep -c -v '^#' {input}) -eq 0 ]]; then
    echo "-> no sv result for: {input}"
    col_n=$(zgrep '#CHROM' {input} | awk '{{print NF}}')
    # CHROM POS ID REF ALT QUAL FILTER INFO FORMAT sample
    dummy_line="X	11111111	.	T	C	.	.	TYPE=DUMMY	GT:SU"
    for i in {{10..$col_n}}; do
        dummy_line="$dummy_line	./.:0"
    done
    printf "$dummy_line" > {output.vcf}.tmp
    gunzip -c {input} | cat - {output.vcf}.tmp > {output.vcf}
    rm {output.vcf}.tmp
else
    gunzip -c {input} > {output.vcf}
fi
gzip -c {output.vcf} > {output.gz}
```


#### lumpy_to_cosmic(30)(split sv of vcf file to gene fusion and other variants)
- rules: var_sv.rules
- input: var_speedseq/{prefix}.sv.vcf.gz
- output: sv/{prefix}.fusion.tsv, sv/{prefix}.fusion.panel.tsv, sv/{prefix}.sv.tsv, sv/{prefix}.sv.panel.tsv
- shell:
```
export PATH=/nfs/pipe/Re/Software/miniconda/bin:/nfs/pipe/Re/Software/bin:$PATH
{lumpy2cosmic} {input} -o sv -b ref/panel.bed.info
```


#### sv_to_cosmic(31)
- rules: var_corr_cosmic.rules
- input: var/{prefix}.sv.vcf
- output: var_corr/{prefix}.sv2cosmic.hit
- shell: /lustre/project/og04/pub/script/sv/sv2cosmic {input} var_corr/{wildcards.prefix} || echo "sv2cosmic run fail: {wildcards.prefix}"


#### plot_panel_depth_samtools(32)
- rules: qc_panel.rules
- input: qc/panel/{sample}.samtools_depth_bed.txt, qc/align/{sample}.samtools_depth.txt.gz
- output: qc/panel/{sample}.panel_depth_samtools.png
- shell: /nfs/pipe/Re/Software/miniconda/bin/perl /lustre/project/og04/pub/pipeline/plot_depth_distribution/depth_distribution.pl -p . -s {wildcards.sample} -o qc/panel -output {wildcards.sample}.panel_depth_samtools -tool samtools


#### bedtools_genomecov_stat(33)
- rules: qc_panel.rules
- input: rules.bedtools_genomecov_bam.output(qc/align/{sample}.bedtools_genomecov.txt)
- output: qc/panel/{sample}.bedtools_genomecov_stat.txt
- shell: 
```
awk ' \
    BEGIN {{ \
        suml1x = 0; sumb1x = 0; \
        suml10x = 0; sumb10x = 0; \
        suml100x = 0; sumb100x = 0; \
        suml1000x = 0; sumb1000x = 0 \
    }} {{ \
        dep = $4; len = $3 - $2; base = dep * len \
    }} {{ \
        if (dep > 0) \
            suml1x += len; sumb1x += base \
    }} {{ \
        if (dep > 9) {{ \
            suml10x += len; sumb10x += base \
        }} \
    }} {{ \
        if (dep > 99) {{ \
            suml100x += len; sumb100x += base \
        }} \
    }} {{ \
        if (dep > 999) {{ \
            suml1000x += len; sumb1000x += base \
        }} \
    }} END {{ \
        print "#total_sum_len_1X:" "\t" suml1x "\t" "total_sum_base_1X:" "\t" sumb1x; \
    print "#total_sum_len_10X:" "\t" suml10x "\t" "total_sum_base_10X:" "\t" sumb10x; \
        print "#total_sum_len_100X:" "\t" suml100x "\t" "total_sum_base_100X:" "\t" sumb100x; \
        print "#total_sum_len_1000X:" "\t" suml1000x "\t" "total_sum_base_1000X:" "\t" sumb1000x; \
    }} \
    ' {input} > {output.total}
```


#### bedtools_intersect(34)
- rules: qc_panel.rules
- input: rules.bedtools_genomecov_bam.output(qc/align/{sample}.bedtools_genomecov.txt)
- output: qc/panel/{sample}.bedtools_intersect.txt
- shell:
```
awk 'FNR==NR {{a[$1]=1;next}} ($1 in a)' <(cut -f 1 {panel_bed} | sort | uniq) {input} > {output}.tmp && \
{bedtools} intersect -a {panel_bed} -b {output}.tmp -wao > {output} && \
rm {output}.tmp
```


#### boxplot_panel_depth(35)
- rules: qc_panel.rules
- input: qc/panel/{sample}.bedtools_intersect.txt
- output: qc/panel/{sample}.boxplot_panel_depth.png
- shell: sh /lustre/project/og04/pub/pipeline/plot_depth_distribution/panel_depth_boxplot.sh -i {input} -o qc/panel -n {wildcards.sample}.boxplot_panel_depth


#### bedtools_coverage_hist(36)
- rules: qc_panel.rules
- input: qc/panel/{sample}.bedtools_intersect.txt
- output: qc/panel/{sample}.bedtools_coverage_hist.txt
- shell: awk -F "\\t" '{{if ($(NF-1) != "0" && $(NF-1) != ".") print $(NF-4) "\\t" $(NF-3) "\\t" $(NF-2) "\\t" $(NF-1) "\\t" $(NF-0)}}' {input} | {bedtools} coverage -hist -a {panel_bed} -b - > {output}


#### bedtools_intersect_uncov(37)
- rules: qc_panel.rules
- input: qc/panel/{sample}.bedtools_intersect.txt
- output: qc/panel/{sample}.bedtools_intersect_uncov.txt
- shell: grep -v "\\b0\\b.*$" {input} | uniq | {bedtools} intersect -v -a {panel_bed} -b - > {output} || echo 'not exit 0'


#### bedtools_intersect_stat(38)
- rules: qc_panel.rules
- input: qc/panel/{sample}.bedtools_intersect.txt
- output: qc/panel/{sample}.bedtools_intersect_stat.txt
- shell:
```
awk -F "\t" \
    'BEGIN {{ \
        suml1x = 0; sumb1x = 0; \
        suml10x = 0; sumb10x = 0; \
        suml100x = 0; sumb100x = 0; \
        suml1000x = 0; sumb1000x = 0; \
        print "#depth\tlength\tbases"; \
    }} {{ \
        dep = $(NF-1); len = $(NF-0); base = dep * len; \
        print dep "\t" len "\t" base \
    }} {{ \
        if (dep > 0)  \
            suml1x += len; sumb1x += base \
    }} {{ \
        if (dep > 9) {{ \
            suml10x += len; sumb10x += base \
        }} \
    }} {{ \
        if (dep > 99) {{ \
            suml100x += len; sumb100x += base \
        }} \
    }} {{ \
        if (dep > 999) {{ \
            suml1000x += len; sumb1000x += base \
        }} \
    }} END {{ \
        {{ \
            print "#capture_sum_len_1X:\t" suml1x "\tcapture_sum_base_1X:\t" sumb1x; \
            print "#capture_sum_len_10X:" "\t" suml10x "\t" "capture_sum_base_10X:" "\t" sumb10x; \
            print "#capture_sum_len_100X:" "\t" suml100x "\t" "capture_sum_base_100X:" "\t" sumb100x; \
            print "#capture_sum_len_1000X:" "\t" suml1000x "\t" "capture_sum_base_1000X:" "\t" sumb1000x; \
        }} \
    }} \
    ' {input} > {output}
```


#### capture_stat(39)
- rules: qc_panel.rules
- input: 
```
rules.bedtools_intersect.output (qc/panel/{sample}.bedtools_intersect.txt)
rules.bedtools_intersect_stat.output  (qc/panel/{sample}.bedtools_intersect_stat.txt)
rules.bedtools_genomecov_bam.output  (qc/align/{sample}.bedtools_genomecov.txt)
rules.bedtools_genomecov_stat.output  (qc/panel/{sample}.bedtools_genomecov_stat.txt)
rules.bedtools_coverage_hist.output  (qc/panel/{sample}.bedtools_coverage_hist.txt)
rules.bedtools_intersect_uncov.output  (qc/panel/{sample}.bedtools_intersect_uncov.txt)
```
- output: qc/panel/{sample}.capture_stat.txt
- shell: 
```
grep '^\#capture_sum_len' {input.capture} > {output}
grep '^\#total_sum_len' {input.total} >> {output}
cap_len=$(grep '^#capture_sum_len_1X' {input.capture} | cut -f 2)
cap_base=$(grep '^#capture_sum_len_1X' {input.capture} | cut -f 4)
cap_ave_dep="$([[ $cap_base -eq 0 ]] && echo 0 || echo $(($cap_base/$cap_len)))"
cap_ave_dep_cent=$(($cap_ave_dep / 10))
awk -F "\t" -v thre=$cap_ave_dep -v thre_cent=$cap_ave_dep_cent ' \
    BEGIN {{ \
        suml_ave = 0; sumb_ave = 0; \
        suml_ave_cent = 0; sumb_ave_cent = 0; \
        print "#capture_average_depth:" "\t" thre "\t" "capture_average_depth_cent" "\t" thre_cent
    }} {{ \
        dep = $(NF-1); len = $(NF-0); base = dep * len; \
    }} {{ \
        if (dep > thre) {{ \
            suml_ave += len; sumb_ave += base; \
        }} \
    }} {{\
        if (dep > thre_cent) {{ \
            suml_ave_cent += len; sumb_ave_cent += base; \
        }} \
    }} \
    END {{ \
        print "#capture_sum_len_above_average_depth:" "\t" suml_ave "\t" "capture_sum_base_above_average_depth:" "\t" sumb_ave; \
        print "#capture_sum_len_above_average_depth_cent:" "\t" suml_ave_cent "\t" "capture_sum_base_above_average_depth_cent:" "\t" sumb_ave_cent; \
    }} \
    ' {input.intersect} >> {output}
awk -F "\t" -v thre=$cap_ave_dep -v thre_cent=$cap_ave_dep_cent ' \
    BEGIN {{ \
        suml_ave = 0; sumb_ave = 0; \
        suml_ave_cent = 0; sumb_ave_cent = 0; \
    }} {{ \
        dep = $4; len = $3 - $2; base = dep * len \
    }} {{ \
        if (dep > thre) {{ \
            suml_ave += len; sumb_ave += base; \
        }} \
    }} {{\
        if (dep > thre_cent) {{ \
            suml_ave_cent += len; sumb_ave_cent += base; \
        }} \
    }} \
    END {{ \
        print "#total_sum_len_above_average_depth:" "\t" suml_ave "\t" "total_sum_base_above_average_depth:" "\t" sumb_ave; \
        print "#total_sum_len_above_average_depth_cent:" "\t" suml_ave_cent "\t" "total_sum_base_above_average_depth_cent:" "\t" sumb_ave_cent; \
    }} \
    ' {input.genome} >> {output}
echo -e "#region_count_not_covered_100%:\t$(grep -c -v -e '^all' -e '1.0000000$' {input.hist})"  >> {output}
echo -e "#region_count_not_captured:\t$(wc -l {input.uncov} | cut -d ' ' -f1)" >> {output}
```


#### calc_coverage(40)
- rules: qc_covmap.rules
- input: qc/covmap/{sample}.coverage
- output: qc/covmap/{sample}.cov
- shell: 
```
cov=`grep -m 1 '^genome' {input} | cut -f 5`
ret=`perl -e \"print substr (1 - \$cov, 0, 6)\"`
echo \"coverage\t$ret\" > {output}
```


#### combine_cov_map(41)
- rules: qc_covmap.rules
- input: qc/covmap/{sample}.cov, qc/covmap/{sample}.map
- output: qc/covmap/{sample}.covmap
- shell: cat {input} > {output}


#### plot_panel_depth(42)
- rules:qc_panel.rules
- input: qc/panel/{sample}.bedtools_intersect.txt, qc/panel/{sample}.bedtools_intersect.txt
- output: qc/panel/{sample}.panel_depth.png
- shell: /nfs/pipe/Re/Software/miniconda/bin/perl /lustre/project/og04/pub/pipeline/plot_depth_distribution/depth_distribution.pl -p . -s {wildcards.sample} -o qc/panel -output {wildcards.sample}.panel_depth


#### varscan_germline_indel(42)
- rules: var_germline_varscan.rules
- input: align/{germline}.mpileup
- output: var_varscan/{germline}.indel.txt
- shell: 
```
export PATH=/nfs2/pipe/Re/Software/miniconda/bin:$PATH
echo ">> min_dp: {par_min_dp}, min_alt_dp: {par_min_alt_dp}, min_var_freq: {par_min_var_freq}, p_value: {par_p_value}"
/nfs2/pipe/Re/Software/miniconda/bin/varscan pileup2indel {input} --min-coverage {par_min_dp} --min-reads2 {par_min_alt_dp} --min-var-freq {par_min_var_freq} --p-value {par_p_value} > {output}
```


#### varscan_germline_snp(43)
- rules: var_germline_varscan.rules
- input: align/{germline}.mpileup
- output: var_varscan/{germline}.snp.txt
- shell:
```
export PATH=/nfs2/pipe/Re/Software/miniconda/bin:$PATH
echo ">> min_dp: {par_min_dp}, min_alt_dp: {par_min_alt_dp}, min_var_freq: {par_min_var_freq}, p_value: {par_p_value}"
/nfs2/pipe/Re/Software/miniconda/bin/varscan pileup2snp {input} --min-coverage {par_min_dp} --min-reads2 {par_min_alt_dp} --min-var-freq {par_min_var_freq} --p-value {par_p_value} > {output}
```


#### varscan_germline_indel_vcf(44)
- rules: var_germline_varscan.rules
- input: align/{germline}.mpileup
- output: var_varscan/{germline}.indel.vcf
- shell: 
```
export PATH=/nfs2/pipe/Re/Software/miniconda/bin:$PATH
echo ">> min_dp: {par_min_dp}, min_alt_dp: {par_min_alt_dp}, min_var_freq: {par_min_var_freq}, p_value: {par_p_value}"
/nfs2/pipe/Re/Software/miniconda/bin/varscan mpileup2indel {input} --min-coverage {par_min_dp} --min-reads2 {par_min_alt_dp} --min-var-freq {par_min_var_freq} --p-value {par_p_value} --output-vcf 1 > {output}
```


#### varscan_germline_snp_vcf(45)
- rules: var_germline_varscan.rules
- input: align/{germline}.mpileup
- output: var_varscan/{germline}.snp.vcf
- shell:
```
export PATH=/nfs2/pipe/Re/Software/miniconda/bin:$PATH
echo ">> min_dp: {par_min_dp}, min_alt_dp: {par_min_alt_dp}, min_var_freq: {par_min_var_freq}, p_value: {par_p_value}"
{varscan} mpileup2snp {input} --min-coverage {par_min_dp} --min-reads2 {par_min_alt_dp} --min-var-freq {par_min_var_freq} --p-value {par_p_value} --output-vcf 1 > {output}
```

#### varscan_filter_indel(46)
- rules: var_germline_varscan.rules
- input: var_varscan/{prefix}.indel.txt,
- output: var_varscan/{prefix}.indel.filter.txt
- shell:
```
export PATH=/nfs2/pipe/Re/Software/miniconda/bin:$PATH
/nfs2/pipe/Re/Software/miniconda/bin/varscan filter {input.indel} --output-file {output}
```


#### varscan_filter_snp(47)
- rules: var_germline_varscan.rules
- input: var_varscan/{prefix}.snp.txt, var_varscan/{prefix}.indel.txt
- output: var_varscan/{prefix}.snp.filter.txt
- shell:
```
export PATH=/nfs2/pipe/Re/Software/miniconda/bin:$PATH
/nfs2/pipe/Re/Software/miniconda/bin/varscan filter {input.snp} --indel-file {input.indel} --output-file {output}
```


#### indel_to_cosmic_germline(48)
- rules: var_corr_cosmic.rules
- input: var_varscan/{germline}.indel.filter.txt
- output: var_corr/{germline}.indel2cosmic.hit
- shell:
```
{indel2cosmic} var_varscan/{wildcards.germline} var_corr/{wildcards.germline} indel.filter.txt || echo "indel2cosmic run fail: {wildcards.germline}"
mv var_corr/{wildcards.germline}.indel2cosmic var_corr/{wildcards.germline}.indel2cosmic.hit || echo "indel2cosmic mv hit fail: {wildcards.germline}"
mv var_corr/{wildcards.germline}.indel2cosmic.2 var_corr/{wildcards.germline}.indel2cosmic.off || echo "indel2cosmic mv off fail: {wildcards.germline}"
rm var_corr/{wildcards.germline}*[012] || echo "indel2cosmic rm fail: {wildcards.germline}"
```


#### intevar_germline(49)
- rules: var_intevar.rules (在panel_intersect.rules也存在，实际使用的是var_intevar.rules)
- input: var_speedseq/{germline}.var.vcf.gz, var_varscan/{germline}.snp.vcf
- output: var_intevar/{germline}_intevar/{germline}.intevar.germline.done
- shell: 
```
/lustre/project/og04/pub/pipeline/intevar/scripts/pipeline.sh -n {wildcards.germline} -i . -o var_intevar
[[ -f var_intevar/{wildcards.germline}_intevar/output_gename/variant_locations.vcf ]] && echo 'Y' > {output}
```


#### cnv_cnvkit_somatic(50)
- rules: var_cnv.rules
- input: lambda wildcards: 'align/' + config['somatic_pair'][wildcards.somatic][0] + '.bam', lambda wildcards: 'align/' + config['somatic_pair'][wildcards.somatic][1] + '.bam'
- output: cnv/{somatic}/{somatic}.cns, cnv/{somatic}/{somatic}.cnr, cnv/{somatic}/{somatic}.gainloss.tsv, cnv/{somatic}/{somatic}.cnv.tsv
- shell:
```
/lustre/project/og04/pub/script/cnv/cnvkit -m somatic -1 '{input.bam_n}' -2 '{input.bam_t}' -o cnv/{wildcards.somatic} -b {panel_bed} -f {fa} -t {thread}
```


#### snv_somatic_speedseq(51)
- rules: var_somatic_speedseq.rules
- input: lambda wildcards: "align/" + config["somatic_pair"][wildcards.somatic][0] + ".bam", lambda wildcards: "align/" + config["somatic_pair"][wildcards.somatic][1] + ".bam"
- output: var_speedseq/{somatic}.var.vcf.gz
- shell:
```
echo ">> min_alt_dp: {par_min_alt_dp}, min_var_freq: {par_min_var_freq}"
/lustre/project/og04/pub/biosoft/speedseq/bin/speedseq somatic -C {par_min_alt_dp} -F {par_min_var_freq} -v -t {thread} -o var_speedseq/{wildcards.somatic}.var {fa} {input.B_N} {input.B_T}
```


#### snv_somatic_mutect(52)
- rules: var_somatic_mutect.rules(24)(60行也有这样一个rule, 实际使用的是60行的)
- input: lambda wildcards: "align/" + config["somatic_pair"][wildcards.somatic][0] + ".bam", lambda wildcards: "align/" + config["somatic_pair"][wildcards.somatic][1] + ".bam"
- output: var_mutect/{somatic}.snv.mutect.txt, var_mutect/{somatic}.snv.mutect.vcf
- shell:
```
echo ">> min_var_freq: {par_min_var_freq}"
/lustre/project/og04/pub/script/var/mutect -r {fa} -1 {input.bam_normal} -2 {input.bam_tumor} -o var_mutect/ -M '--fraction_contamination {par_min_var_freq} --minimum_mutation_cell_fraction {par_min_var_freq} --tumor_f_pretest {par_min_var_freq}'
```


#### sv_somatic_speedseq(53)
- rules: var_somatic_speedseq.rules
- input: 
```
lambda wildcards: "align/" + config["somatic_pair"][wildcards.somatic][0] + ".bam",
lambda wildcards: "align/" + config["somatic_pair"][wildcards.somatic][1] + ".bam",
lambda wildcards: "align/" + config["somatic_pair"][wildcards.somatic][0] + ".splitters.bam",
lambda wildcards: "align/" + config["somatic_pair"][wildcards.somatic][1] + ".splitters.bam",
lambda wildcards: "align/" + config["somatic_pair"][wildcards.somatic][0] + ".discordants.bam",
lambda wildcards: "align/" + config["somatic_pair"][wildcards.somatic][1] + ".discordants.bam",
```
- output: var_speedseq/{somatic}.sv.vcf.gz
- shell: /lustre/project/og04/pub/biosoft/speedseq/bin/speedseq sv -v -x {speedseq_bed}/lumpy.exclude.bed -t {thread} {params.custom} -o var_speedseq/{wildcards.somatic} -R {fa} -B {input.B_N},{input.B_T} -S {input.S_N},{input.S_T} -D {input.D_N},{input.D_T} || echo '-> no sv'  # to avoid non-zero exit status due to none sv called



#### dups_summary(54)
- rules: qc_align.rules
- input: qc/align/{sample}.samtools_dups.txt", sample = config['sample']
- output: qc/align/dups_summary.txt
- shell:
```
echo -e "#sample\tsamtools_dups\tinsert_size" > qc/align/dups_summary.txt
for i in qc/align/*.samtools_dups.txt; do
    sample=$(basename $i .samtools_dups.txt)
    dups=$(cat $i)
    insert_size=$(grep 'insert size average' qc/align/$sample.samtools_stat.txt | cut -f3)
    echo -e "$sample\t$dups\t$insert_size" >> qc/align/dups_summary.txt
done
```

#### summary_filterstat(55)
- rules: filter_raw.rules
- input: clean/{sample}_filterstat.txt", sample = config["sample"], clean/{sample}_filter.log", sample = config["sample"]
- output: qc/filter/filterstat_summary.txt
- shell: 
```
"cp -v {input} qc/filter;"\
"{summary_filterstat} qc/filter"
```


#### multiqc_bam(56)
- rules: qc_align.rules
- input: qc/align/{sample}.fastqc.html", sample = config['sample']
- output: qc/align/multiqc_report.html
- shell:
```
cd qc/align
/nfs2/biosoft/bin/multiqc -f .
```


#### summary_covmap(57)
- rules: qc_covmap.rules
- input: qc/covmap/{sample}.covmap", sample = config["sample"]
- output: qc/covmap/covmap_summary.txt
- shell:
```
/lustre/project/og04/pub/script/summary_covmap qc/covmap
```


#### summary_fqstat_clean(58)
- rules: qc_clean.rules
- input: qc/clean/{sample}_R1_fqstat.txt", sample = config["sample"], qc/clean/{sample}_R2_fqstat.txt", sample = config["sample"]
- output: qc/clean/fqstat_summary.txt, qc/clean/fqstat_summary_pe.txt
- shell: /lustre/project/og04/pub/script/summary_fqstat qc/clean


#### multiqc_clean(59)
- rules: qc_clean.rules
- input: qc/clean/{sample}_R1_fastqc.html", sample = config["sample"], qc/clean/{sample}_R2_fastqc.html", sample = config["sample"]
- output: qc/clean/multiqc_report.html
- shell:
```
cd qc/clean;\
/nfs2/biosoft/bin/multiqc -f .
```

#### summary_capture_stat(60)
- rules: qc_panel.rules
- input: qc/panel/{sample}.capture_stat.txt", sample = config["sample"], rules.panel_bed_stat.output(qc/panel/panel_bed_stat.txt)
- output: qc/panel/capture_stat_summary.txt
- shell:
```
/lustre/project/og04/pub/script/summary_panelstat qc/panel
```

#### join_qc_summary(61)
- rules: all_qc.rules
- input: 
```
"qc/raw/fqstat_summary_pe.txt",
"qc/filter/filterstat_summary.txt",
"qc/clean/fqstat_summary_pe.txt",
"qc/covmap/covmap_summary.txt",
"qc/panel/capture_stat_summary.txt",
"qc/panel/panel_bed_stat.txt",
"qc/align/dups_summary.txt",
```
- output:qc/qc_summary.txt
- shell:
```
panel_size=$(grep 'region_len' qc/panel/panel_bed_stat.txt | cut -f 2)

/nfs/pipe/Re/Software/bin/csvjoin -t -c 1 qc/raw/fqstat_summary_pe.txt qc/filter/filterstat_summary.txt qc/clean/fqstat_summary_pe.txt qc/covmap/covmap_summary.txt qc/panel/capture_stat_summary.txt qc/align/dups_summary.txt | {csvformat} -T | awk -v panel_size=$panel_size 'FS="\t" {{if ($0 ~ "#") print $0, "\t", "seq_depth", "\t", "trim_adapter"; else {{total=$2*$3; trim_perc=1-$4/total; print $0, "\t", $4/panel_size, "\t", trim_perc}} }}' | sed -e 's/\ \t/\t/g' -e 's/\t\ /\t/g' > {output}

/nfs/pipe/Re/Software/bin/csvcut -t -c '#sample,5,6,10,12,13,low_qual_filter(%),adapter_filter(%),undersize_ins_filter(%),duplicated_filter(%),29,30,34,36,37,coverage,mapping_rate,coverage_cent,specificity_cent,uniformity_cent,capture_depth_1X,samtools_dups,insert_size,seq_depth,trim_adapter' {output} | {csvformat} -T > qc/qc_summary_brief.txt
```

#### intevar_vcf_to_filter_vcf_germline(62)
- rules: var_intevar.rules
- input: var_intevar/{germline}_intevar/{germline}.intevar.germline.done, qc/qc_summary.txt
- output: var_intevar/{germline}_intevar/{germline}.var.filter.vcf
- shell:
```
echo "> processe on: {wildcards.germline}"
# dep=$(awk '{{if ($1 == "{wildcards.germline}") print $NF}}' {input.qc})
dep=$({csvgrep} -t -m '{wildcards.germline}' -c '#sample' {input.qc} | {csvcut} -c 'capture_depth_1X' | tail -1)
dep_thre=$(echo "$dep * {dep_thre_percent_germline}" | /usr/bin/bc)
dep_thre=${{dep_thre%%\.*}}
echo ">> dep: $dep, dep_thre_percent: {dep_thre_percent_germline}, dep_thre: $dep_thre"
[[ -f {output} ]] && echo "! output exists, will remove" && rm -v {output}.tmp
[[ -f {output}.tmp ]] && echo "! output.tmp exists, will remove" && rm -v {output}.tmp
[[ -f {output}.header ]] && echo "! output.header exists, will remove" && rm -v {output}.header
while read line; do
    if [[ ${{line:0:1}} == '#' ]]; then
        printf "$line\n" >> {output}.header
    else
        dp=${{line##*,dp=}}
        dp=${{dp%%;}}
        if [[ $dp -gt $dep_thre ]]; then
            printf "$line\n" >> {output}.tmp
        fi
    fi
done < var_intevar/{wildcards.germline}_intevar/output_gename/variant_locations.vcf
sort -k1 -V {output}.tmp | uniq | sed 's/^chr//g' > {output}.body
cat {output}.header {output}.body > {output}.all
cat {output}.header > {output}
{bedtools} intersect -a {output}.all -b {panel_bed} >> {output}
rm {output}.tmp {output}.header {output}.body {output}.all
echo ">> input: $(wc -l var_intevar/{wildcards.germline}_intevar/output_gename/variant_locations.vcf)"
echo ">> output: $(wc -l {output})"
```

#### intevar_filter_vcf_to_transvar_vcf_panel_germline(63)
- rules: var_intevar.rules
- input: var_intevar/{germline}_intevar/{germline}.var.filter.vcf
- output: var_intevar/{germline}_intevar/{germline}.var.filter.panel.vcf, var_intevar/{germline}_intevar/{germline}.var.filter.panel.transvar.vcf
- shell:
```
echo "> processe on: {wildcards.germline}"
/nfs2/pipe/Re/Software/miniconda/bin/bedtools intersect -a {input} -b ref/panel.bed.info | uniq > {output.vcf}
/nfs/pipe/Re/Software/bin/transvar ganno --vcf {output.vcf} --ensembl --longest > {output.transvar}
```

#### intevar_transvar_vcf_to_bed_panel_germline(64)
- rules: var_intevar.rules
- input: var_intevar/{germline}_intevar/{germline}.var.filter.panel.transvar.vcf
- output: var_intevar/{germline}_intevar/{germline}.var.filter.panel.transvar.bed
- shell:
```
export PATH=/nfs/pipe/Re/Software/miniconda/bin:$PATH
/nfs/pipe/Re/Software/miniconda/bin/convert2bed -i vcf -o bed < {input} > {output}
```


#### intevar_transvar_bed_format_panel_germline(65)
- rules: var_intevar.rules
- input: var_intevar/{germline}_intevar/{germline}.var.filter.panel.transvar.bed
- output: var_intevar/{germline}.var.panel.tsv
- shell:
```
echo -e "#CHROM\tPOS\tEND\tID\tQUAL\tREF\tALT\tFILTER\ttranscript\tgene\tstrand\tganno\tcanno\tpanno\tregion\tinfo\tVF\tDP\tDP_ALT" > {output.panel_bed}.tmp
sed -e 's/^chr//g' -e 's/{wildcards.germline}:.*,vf=//g' -e 's/,dp=/\t/g' -e 's/;\t/\t/g' -e 's|\tchr.*:\(g\.\S*\)/\(\S*\)/\(\S*\)\t|\t\\1\t\\2\t\\3\t|g' {input} | awk -F "\t" '{{printf \"%s\\t%s\\t%s\\t%s\\t%s\\t%s\\t%s\\t%s\\t%s\\t%s\\t%s\\t%s\\t%s\\t%s\\t%s\\t%s\\t%s\\t%s\\t%.f\\n\", $1,$2,$3,$4,$5,$6,$7,$8, $11,$12,$13,$14,$15,$16,$17,$18, $9,$10, $9 * $10 }}' >> {output.panel_bed}.tmp
[[ ! -f var_intevar/panel_bed_info.header ]] && head -1 ref/panel.bed.info > var_intevar/panel_bed_info.header
head -1 {output.panel_bed}.tmp > {output.panel_bed}.tmp.header
paste var_intevar/panel_bed_info.header {output.panel_bed}.tmp.header > {output.panel_bed}
{bedtools} intersect -a ref/panel.bed.info -b {output.panel_bed}.tmp -wa -wb >> {output.panel_bed}
rm -vf {output.panel_bed}.tmp.header {output.panel_bed}.tmp
```

#### report_chemo(66)
- rules: report.rules
- input: var_intevar/{germline}.var.panel.tsv
- output: var_intevar/{germline}.var.panel.tsv
- shell: {python} {report_chemo} -i {input} -o report/{wildcards.germline} || touch {output}


#### varscan_somatic_copynumber(67)
- rules: var_somatic_varscan.rules
- input: lambda wildcards: "align/" + config["somatic_pair"][wildcards.somatic][0] + ".mpileup", lambda wildcards: "align/" + config["somatic_pair"][wildcards.somatic][1] + ".mpileup"
- output: var_varscan/{somatic}.copynumber.txt
- shell:
```
export PATH=/nfs2/pipe/Re/Software/miniconda/bin:$PATH
echo ">> min_dp: {par_min_dp}, min_alt_dp: {par_min_alt_dp}, min_var_freq: {par_min_var_freq}, p_value: 0.01 (fix)"
{varscan} copynumber {input.normal} {input.tumor} {output} --min-coverage {par_min_dp} --min-reads2 {par_min_alt_dp} --min-var-freq {par_min_var_freq} --p-value 0.01
mv {output}.copynumber {output}
```


####  varscan_somatic_var(68)
- rules: var_somatic_varscan.rules
- input: lambda wildcards: "align/" + config["somatic_pair"][wildcards.somatic][0] + ".mpileup", lambda wildcards: "align/" + config["somatic_pair"][wildcards.somatic][1] + ".mpileup"
- output: var_varscan/{somatic}.snv.txt, var_varscan/{somatic}.sindel.txt
- shell:
```
export PATH=/nfs2/pipe/Re/Software/miniconda/bin:$PATH
echo ">> min_dp: {par_min_dp}, min_alt_dp: {par_min_alt_dp}, min_var_freq: {par_min_var_freq}, p_value: {par_p_value}"
{varscan} somatic {input.normal} {input.tumor} --min-coverage {par_min_dp} --min-reads2 {par_min_alt_dp} --min-var-freq {par_min_var_freq} --p-value {par_p_value} --output-snp {output.snp} --output-indel {output.indel}
```

#### varscan_somatic_var_vcf(69)
- rules: var_somatic_varscan.rules
- input: lambda wildcards: "align/" + config["somatic_pair"][wildcards.somatic][0] + ".mpileup", lambda wildcards: "align/" + config["somatic_pair"][wildcards.somatic][1] + ".mpileup"
- output: var_varscan/{somatic}.snv.vcf, var_varscan/{somatic}.sindel.vcf
- shell:
```
export PATH=/nfs2/pipe/Re/Software/miniconda/bin:$PATH
echo ">> min_dp: {par_min_dp}, min_alt_dp: {par_min_alt_dp}, min_var_freq: {par_min_var_freq}, p_value: {par_p_value}"
{varscan} somatic {input.normal} {input.tumor} --min-coverage {par_min_dp} --min-reads2 {par_min_alt_dp} --min-var-freq {par_min_var_freq} --p-value {par_p_value} --output-snp {output.snp} --output-indel {output.indel} --output-vcf 1
```


####  varscan_somatic_copycaller(70)
- rules:var_somatic_varscan.rules
- input: rules.varscan_somatic_copynumber.output (var_varscan/{somatic}.copynumber.txt)
- output: var_varscan/{somatic}.copynumber.filter.txt, var_varscan/{somatic}.copynumber.homdel.txt
- shell:
```
export PATH=/nfs2/pipe/Re/Software/miniconda/bin:$PATH
{varscan} copyCaller {input} --output-file {output.call} --output-homdel-file {output.homdel}
```


#### varscan_somatic_filter_indel(71)
- rules: var_somatic_varscan.rules
- input: var_varscan/{somatic}.snv.txt, var_varscan/{somatic}.sindel.txt
- output: var_varscan/{somatic}.sindel.filter.txt
- shell: 
```
export PATH=/nfs2/pipe/Re/Software/miniconda/bin:$PATH
echo ">> min_dp: {par_min_dp}, min_alt_dp: {par_min_alt_dp}, min_var_freq: {par_min_var_freq}, p_value: {par_p_value}"
{varscan} somaticFilter {input.indel} --min-coverage {par_min_dp} --min-reads2 {par_min_alt_dp} --min-var-freq {par_min_var_freq} --p-value {par_p_value} --output-file {output}
```

#### varscan_somatic_filter_snp(72)
- rules: var_somatic_varscan.rules
- input: var_varscan/{somatic}.snv.txt, var_varscan/{somatic}.sindel.txt
- output: var_varscan/{somatic}.snv.filter.txt
- shell:
```
export PATH=/nfs2/pipe/Re/Software/miniconda/bin:$PATH
echo ">> min_dp: {par_min_dp}, min_alt_dp: {par_min_alt_dp}, min_var_freq: {par_min_var_freq}, p_value: {par_p_value}"
{varscan} somaticFilter {input.snp} --min-coverage {par_min_dp} --min-reads2 {par_min_alt_dp} --min-var-freq {par_min_var_freq} --p-value {par_p_value} --indel-file {input.indel} --output-file {output}
```


#### varscan_somatic_process_snp(73)
- rules: var_somatic_varscan.rules
- input: var_varscan/{somatic}.snv.txt
- output:
```
"var_varscan/{somatic}.snv.txt.Germline",
"var_varscan/{somatic}.snv.txt.Germline.hc",
"var_varscan/{somatic}.snv.txt.Somatic",
"var_varscan/{somatic}.snv.txt.Somatic.hc",
"var_varscan/{somatic}.snv.txt.LOH",
"var_varscan/{somatic}.snv.txt.LOH.hc",
```
- shell:
```
export PATH=/nfs2/pipe/Re/Software/miniconda/bin:$PATH
{varscan} processSomatic {input}
```


#### varscan_somatic_process_indel(74)
- rules: var_somatic_varscan.rules
- input: var_varscan/{somatic}.sindel.txt
- output:
```
"var_varscan/{somatic}.sindel.txt.Germline",
"var_varscan/{somatic}.sindel.txt.Germline.hc",
"var_varscan/{somatic}.sindel.txt.Somatic",
"var_varscan/{somatic}.sindel.txt.Somatic.hc",
"var_varscan/{somatic}.sindel.txt.LOH",
"var_varscan/{somatic}.sindel.txt.LOH.hc",
```
- shell:
```
export PATH=/nfs2/pipe/Re/Software/miniconda/bin:$PATH
{varscan} processSomatic {input}
```


#### indel_to_cosmic_somatic(75)
- rules: var_corr_cosmic.rules 
- input: var_varscan/{somatic}.sindel.filter.txt
- output: var_corr/{somatic}.sindel2cosmic.hit
- shell:
```
{indel2cosmic} var_varscan/{wildcards.somatic} var_corr/{wildcards.somatic} sindel.filter.txt || echo "indel2cosmic run fail: {wildcards.somatic}"
mv var_corr/{wildcards.somatic}.indel2cosmic var_corr/{wildcards.somatic}.sindel2cosmic.hit || echo "indel2cosmic mv hit fail: {wildcards.somatic}"
mv var_corr/{wildcards.somatic}.indel2cosmic.2 var_corr/{wildcards.somatic}.sindel2cosmic.off || echo "indel2cosmic mv off fail: {wildcards.somatic}"
rm var_corr/{wildcards.somatic}*[012] || echo "indel2cosmic rm fail: {wildcards.somatic}"
```


#### intevar_somatic(76)
- rules: var_intevar.rules
- input: var_speedseq/{somatic}.var.vcf.gz, var_varscan/{somatic}.snv.vcf, var_mutect/{somatic}.snv.mutect.vcf, var_mutect/{somatic}.snv.mutect.txt
- output: var_intevar/{somatic}_intevar/{somatic}.intevar.somatic.done
- shell:
```
{intevar} -n {wildcards.somatic} -i . -o var_intevar
[[ -f var_intevar/{wildcards.somatic}_intevar/output_gename/variant_locations.vcf ]] && echo 'Y' > {output}
```


#### intevar_vcf_to_filter_vcf_somatic(77)
- rules: var_intevar.rules
- input: var_intevar/{somatic}_intevar/{somatic}.intevar.somatic.done, qc/qc_summary.txt
- output: var_intevar/{somatic}_intevar/{somatic}.svar.filter.vcf
- shell:
```
echo "> processe on: {wildcards.somatic}"
somatic_pair={wildcards.somatic}
normal=${{somatic_pair%%-VS-*}}
tumor=${{somatic_pair##*-VS-}}
echo "> somatic: $somatic_pair, normal: $normal, tumor: $tumor"
# dep_normal=$(awk -v normal=$normal '{{if ($1 == normal) print $NF}}' {input.qc})
dep_normal=$({csvgrep} -t -m $normal -c '#sample' {input.qc} | {csvcut} -c 'capture_depth_1X' | tail -1)
# dep_tumor=$(awk -v tumor=$tumor '{{if ($1 == tumor) print $NF}}' {input.qc})
dep_tumor=$({csvgrep} -t -m $tumor -c '#sample' {input.qc} | {csvcut} -c 'capture_depth_1X' | tail -1)
dep_normal=${{dep_normal%%\.*}}
dep_tumor=${{dep_tumor%%\.*}}
[[ $dep_normal -lt $dep_tumor ]] && dep="$dep_normal" || dep="$dep_tumor"
dep_thre=$(echo "$dep * {dep_thre_percent_somatic}" | /usr/bin/bc)
dep_thre=${{dep_thre%%\.*}}
echo ">> dep_normal: $dep_normal, dep_tumor: $dep_tumor"
echo ">> dep: $dep, dep_thre_percent: {dep_thre_percent_somatic}, dep_thre: $dep_thre"
[[ -f {output} ]] && echo "! output exists, will remove" && rm -v {output}
[[ -f {output}.tmp ]] && echo "! output.tmp exists, will remove" && rm -v {output}.tmp
[[ -f {output}.header ]] && echo "! output.header exists, will remove" && rm -v {output}.header
while read line; do
    if [[ ${{line:0:1}} == '#' ]]; then
        printf "$line\n" >> {output}.header
    else
        dp=${{line##*,dp=}}
        dp=${{dp%%;}}
        if [[ $dp -gt $dep_thre ]]; then
            printf "$line\n" >> {output}.tmp
        fi
    fi
done < var_intevar/{wildcards.somatic}_intevar/output_gename/variant_locations.vcf
sort -k1 -V {output}.tmp | uniq | sed 's/^chr//g' > {output}.body
cat {output}.header {output}.body > {output}.all
cat {output}.header > {output}
{bedtools} intersect -a {output}.all -b {panel_bed} >> {output}
rm {output}.tmp {output}.header {output}.body {output}.all
echo ">> input: $(wc -l var_intevar/{wildcards.somatic}_intevar/output_gename/variant_locations.vcf)"
echo ">> output: $(wc -l {output})"
```


#### intevar_filter_vcf_to_transvar_vcf_panel_somatic(78)
- rules: var_intevar.rules
- input: var_intevar/{somatic}_intevar/{somatic}.svar.filter.vcf
- output: var_intevar/{somatic}_intevar/{somatic}.svar.filter.panel.vcf, var_intevar/{somatic}_intevar/{somatic}.svar.filter.panel.transvar.vcf
- shell:
```
echo "> processe on: {wildcards.somatic}"
{bedtools} intersect -a {input} -b ref/panel.bed.info | uniq > {output.vcf}
{transvar} ganno --vcf {output.vcf} --ensembl --longest > {output.transvar}
```


#### intevar_transvar_vcf_to_bed_panel_somatic(79)
- rules: var_intevar.rules
- input: var_intevar/{somatic}_intevar/{somatic}.svar.filter.panel.transvar.vcf
- output: var_intevar/{somatic}_intevar/{somatic}.svar.filter.panel.transvar.bed
- shell:
```
export PATH=/nfs/pipe/Re/Software/miniconda/bin:$PATH
{convert2bed} -i vcf -o bed < {input} > {output}
```


#### intevar_transvar_bed_format_panel_somatic(80)
- rules: var_intevar.rules
- input: var_intevar/{somatic}_intevar/{somatic}.svar.filter.panel.transvar.bed
- output: var_intevar/{somatic}.svar.panel.tsv
- shell: 
```
echo -e "#CHROM\tPOS\tEND\tID\tQUAL\tREF\tALT\tFILTER\ttranscript\tgene\tstrand\tganno\tcanno\tpanno\tregion\tinfo\tVF\tDP\tDP_ALT" > {output.panel_bed}.tmp
sed -e 's/^chr//g' -e 's/{wildcards.somatic}:.*,vf=//g' -e 's/,dp=/\t/g' -e 's/;\t/\t/g' -e 's|\tchr.*:\(g\.\S*\)/\(\S*\)/\(\S*\)\t|\t\\1\t\\2\t\\3\t|g' {input} | awk -F "\t" '{{printf \"%s\\t%s\\t%s\\t%s\\t%s\\t%s\\t%s\\t%s\\t%s\\t%s\\t%s\\t%s\\t%s\\t%s\\t%s\\t%s\\t%s\\t%s\\t%.f\\n\", $1,$2,$3,$4,$5,$6,$7,$8, $11,$12,$13,$14,$15,$16,$17,$18, $9,$10, $9 * $10 }}' >> {output.panel_bed}.tmp
[[ ! -f var_intevar/panel_bed_info.header ]] && head -1 ref/panel.bed.info > var_intevar/panel_bed_info.header
head -1 {output.panel_bed}.tmp > {output.panel_bed}.tmp.header
paste var_intevar/panel_bed_info.header {output.panel_bed}.tmp.header > {output.panel_bed}
{bedtools} intersect -a ref/panel.bed.info -b {output.panel_bed}.tmp -wa -wb >> {output.panel_bed}
rm -vf {output.panel_bed}.tmp.header {output.panel_bed}.tmp
```


#### cp_output(81)
- rules: pipe_tumor.rules
- input: 
```
# qc
"qc/qc_summary.txt",
# var_intevar
expand('var_intevar/{germline}.var.panel.tsv', germline = config['germline']),
expand('var_intevar/{somatic}.svar.panel.tsv', somatic = config['somatic']),
# var_corr
expand("var_corr/{germline}.indel2cosmic.hit", germline = config['germline']),
expand("var_corr/{germline}.sv2cosmic.hit", germline = config['germline']),
expand("var_corr/{somatic}.sindel2cosmic.hit", somatic = config['somatic']),
expand("var_corr/{somatic}.sv2cosmic.hit", somatic = config['somatic']),
# anno_process
#expand('anno_process/{germline}.var.join.tsv', germline = config['germline']),
#expand('anno_process/{germline}.sv.join.tsv', germline = config['germline']),
#expand('anno_process/{somatic}.var.join.tsv', somatic = config['somatic']),
#expand('anno_process/{somatic}.sv.join.tsv', somatic = config['somatic']),
#expand('anno_process/{germline}.var.join.freq.panel.tsv', germline = config['germline']),
#expand('anno_process/{germline}.sv.join.freq.panel.tsv', germline = config['germline']),
#expand('anno_process/{somatic}.var.join.freq.panel.tsv', somatic = config['somatic']),
#expand('anno_process/{somatic}.sv.join.freq.panel.tsv', somatic = config['somatic']),
# cnvkit
expand('cnv/{somatic}/{somatic}.cns', somatic = config['somatic']),
expand('cnv/{germline}/{germline}.cns', germline = config['germline']),
#var_sv:
expand('sv/{germline}.fusion.factera.log', germline = config['germline']),
expand('sv/{germline}.fusion.panel.tsv', germline = config['germline']),
expand('sv/{somatic}.fusion.panel.tsv', somatic = config['somatic']),
expand('sv/{germline}.sv.panel.tsv', germline = config['germline']),
expand('sv/{somatic}.sv.panel.tsv', somatic = config['somatic']),
# purity
#'advance/purity_summary.txt',
# report
expand('report/{germline}.docx', germline = config['germline']),
```
- output: output/{prefix}.log
- shell:
```
echo "START cp @$(date)" > {output.logs}
cp -vf project_info.txt output >> {output.logs} || echo ""
echo "> qc" >> {output.logs}
cp -vf qc/qc_summary*.txt output >> {output.logs} || echo ""
echo "> var" >> {output.logs}
cp -vf var_intevar/*var.tsv output >> {output.logs} || echo ""
cp -vf var_intevar/*var.panel.tsv output >> {output.logs} || echo ""
echo "> sv" >> {output.logs}
cp -vf sv/*.sv.*tsv output >> {output.logs} || echo ""
echo "> fusion" >> {output.logs}
cp -vf sv/*.fusion.*tsv output >> {output.logs} || echo ""
echo "> cnv" >> {output.logs}
find cnv -name '*.cnv.tsv' | xargs -I {{}} cp -vf {{}} output >> {output.logs} || echo ""
echo "> report" >> {output.logs}
cp -vf report/*.docx output >> {output.logs} || echo ""
find output -name '*.docx' -size 0 | xargs rm -v >> {output.logs} || echo "no empty report" >> {output.logs}
echo "> rm" >> {output.logs}
find output -name '*factera*' -size 0 | xargs rm -v >> {output.logs} || echo "no empty factera" >> {output.logs}
echo "DONE cp @$(date)" >> {output.logs}
```

#### all_pipe_tumor(82)
- rules: pipe_tumor.rules