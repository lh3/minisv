## Getting Started
```sh
# Download k8 v1.0; better copy "k8" to a directory in PATH
curl -L https://zenodo.org/records/8245120/files/k8-1.0.tar.bz2?download=1 | tar -jxf -
cp k8-1.0/k8-`uname -m`-`uname -s` k8

# alignment (requiring minigraph-0.21+ and minimap2-2.28+)
minimap2 -cxmap-hifi -s50 -t16 --ds hg38.fa hifi.fq.gz > hs38l.paf
minimap2 -cxmap-hifi -s50 -t16 --ds chm13v2.fa hifi.fq.gz > chm13l.paf
minigraph -cxlr -t16 chm13-90c.r518.gfa.gz hifi.fq.gz > chm13g.paf

# extract candidate SVs (this may take >10 minutes on real data)
k8 minisv.js easyjoin -c "k8 minisv.js" -n tumor hs38l.paf chm13l.paf chm13g.paf | bash > join.msv

# generate SV calls
sort -k1,1 -k2,2n -S4G join.msv | k8 minisv.js merge - > call.msv
k8 minisv.js genvcf call.msv > call.vcf

# tumor-normal pairs
k8 minisv.js extract -n normal normal.hs38l.paf > normal.msv
cat join.msv normal.msv | sort -k1,1 -k2,2n -S4G | k8 minisv.js merge - \
  | grep tumor | grep -v normal > tumor.msv
```

## Introduction

Minisv is a lightweight structural variation (SV) caller for long genomic
reads. While minisv can call germline SVs, it is primarily optimized for calling 
mosaic SVs, somatic SVs in paired or unpaired tumor samples, or de novo SVs.
The basic idea behind minisv is to use the read aligned to different references
to greatly reduce alignment errors.
