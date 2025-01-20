"""
# Up Lifting

```shell
awk '{print $1"\t"$2"\t"$3"\t"NR}' ../TangZ_etal.Cell2015.ChIA-PET_GM12878_CTCF.published_PET_clusters.no_black.txt > pet1.bed
awk '{print $4"\t"$5"\t"$6"\t"NR}' ../TangZ_etal.Cell2015.ChIA-PET_GM12878_CTCF.published_PET_clusters.no_black.txt > pet2.bed
```

Using 'https://genome.ucsc.edu/cgi-bin/hgLiftOver' produce `lift1.bed` and `lift2.bed` with parameters:
```
Minimum ratio of bases that must map:	0.95
 
BED 4 to BED 6 Options
Allow multiple output regions:	off
  Minimum hit size in query:	0
  Minimum chain size in target:	0
 
BED 12 Options
Min ratio of alignment blocks or exons that must map:	1.00
If thickStart/thickEnd is not mapped, use the closest mapped base:	off
```


```shell
awk '{print $1"\t"$2"\t"$3"\t"NR}' ../wgEncodeAwgDnaseUwdukeGm12878UniPk.narrowPeak > dnase1.bed
awk '{print $1"\t"$2"\t"$3"\t"NR}' ../wgEncodeAwgTfbsBroadGm12878CtcfUniPk.narrowPeak.txt > peaks1.bed
```
Then produce `lift_dnase.bed` and `lift_peaks.bed` using `dnase1.bed` and `dnase2.bed` with same parameters except `Allow multiple output regions:	on`

Then run to join original rows with lifted data:
```shell
python join_lifted.py 
python join_lifted.py peaks
python join_lifted.py dnase
```
"""