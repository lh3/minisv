## Getting Started
```sh
# Download k8 v1.0; better copy "k8" to a directory in PATH
curl -L https://zenodo.org/records/8245120/files/k8-1.0.tar.bz2?download=1 | tar -jxf -
cp k8-1.0/k8-`uname -m`-`uname -s` k8

# extract candidate SVs
k8 minisv.js ej -c "k8 minisv.js" -n COLO829T test/COLO829T.hs38l.paf.gz \
  test/COLO829T.chm13g.paf.gz | bash > COLO829T.rsv
k8 minisv.js extract -n COLO829BL test/COLO829BL.hs38l.paf.gz > COLO829BL.rsv

# single-sample calling
sort -k1,1 -k2,2n -S4G COLO829T.rsv | k8 minisv.js merge - > COLO829T.msv
k8 minisv.js genvcf COLO829T.msv > COLO829T.vcf

# paired calling
sort -k1,1 -k2,2n -S4G COLO829{T,BL}.rsv | k8 minisv.js merge - \
  | grep COLO829T | grep -v COLO829BL > COLO829T.paired.msv
```

## Introduction

Minisv is a lightweight structural variation (SV) caller for long genomic
reads. It is primarily developed for calling mosaic SVs, somatic SVs in paired
or unpaired tumor samples, or de novo SVs. While minisv can also call germline
SVs, existing germline callers will generally do better.

The key advantage of minisv is to combine read alignments against multiple
references, including pangenome graphs and the assembly of input reads. Minisv
calls a SV only if it can be observed in all alignment. This simple strategy
reduces alignment errors and filters out germline SVs in the sample (when used
with the assembly of input reads) or in the population (when used with
pangenomes).

For PacBio HiFi reads at high coverage, minisv achieves much higher specificity
in mosaic SV calling, has comparable accuracy to tumor-normal paired SV callers,
and is the only tool accurate enough for calling large-scale chromosomal
alterations from a single tumor sample without matched normal.

## Usage
