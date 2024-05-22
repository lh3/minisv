## <a name="start"></a>Getting Started
```sh
# Download k8 v1.0; better copy "k8" to a directory in PATH
curl -L https://zenodo.org/records/8245120/files/k8-1.0.tar.bz2?download=1 | tar -jxf -
cp k8-1.0/k8-`uname -m`-`uname -s` k8

# extract candidate SVs
k8 minisv.js e -c "k8 minisv.js" -n COLO829T test/COLO829T.hs38l.paf.gz \
  test/COLO829T.chm13g.paf.gz | bash > COLO829T.rsv
k8 minisv.js extract -n COLO829BL test/COLO829BL.hs38l.paf.gz > COLO829BL.rsv

# single-sample calling
sort -k1,1 -k2,2n -S4G COLO829T.rsv | k8 minisv.js merge - > COLO829T.msv
k8 minisv.js genvcf COLO829T.msv > COLO829T.vcf

# paired calling
sort -k1,1 -k2,2n -S4G COLO829{T,BL}.rsv | k8 minisv.js merge - \
  | grep COLO829T | grep -v COLO829BL > COLO829T.paired.msv
```

## Table of Contents

- [Getting Started](#start)
- [Introduction](#intro)
- [Workflow](#design)
- [Calling SVs](#call-sv)
  - [Germline SVs](#call-germline)
  - [Somatic SVs in tumor-normal pairs](#call-pair)
  - [Large somatic SVs in tumor-only samples](#call-tonly)
  - [Mosaic SVs](#call-mosaic)
- [Comparing SVs](#compare)
- [Limitations](#limit)

## <a name="intro"></a>Introduction

Minisv is a lightweight mosaic/somatic structural variation (SV) caller for
long genomic reads. Different from other SV callers, it prefers to combine read
alignments against multiple reference genomes. Minisv retains an SV on a read
only if the SV is observed on the read alignments against all references. This
simple strategy reduces alignment errors and filters out germline SVs in the
sample (when used with the assembly of input reads) or in the population (when
used with pangenomes).

Given PacBio HiFi reads at high coverage, minisv
achieves higher specificity for mosaic SV calling, has comparable accuracy to
tumor-normal paired SV callers, and is the only tool accurate enough for
calling large chromosomal alterations (CAs) from a single tumor sample without
matched normal.

## <a name="design"></a>Workflow

Minisv can call 1) germline SVs, 2) somatic SVs in tumor-normal pairs, 3) large
somatic SVs in a single tumor, 4) mosaic SVs and 5) de novo SVs in a trio; the
exact use also depends on the input data types. Minisv achieves this variety of
calling modes in three steps:

1. Extract SVs from read alignment with `extract`. This command processes one
   read at a time, extracts long INDELs or breakends and outputs in a BED-like
   minisv format. If you have tumor-normal pairs, remember to use `-n` to
   specify samples such thatdd you can distinguish the samples in step 3.

2. Intersect candidate SVs extracted from multiple alignment files with `isec`.
   You can skip this step if you use one reference genome only, but you would
   lose the key advantage of minisv. The first two steps are shared by all
   calling modes, though the input alignments may be different. You can combine
   the first two steps with the `e` command, which provides a more convenient but
   less flexible interface.

3. Call SVs supported by multiple reads using `merge`. This command counts the
   number of supporting reads for each sample. You can filter out SVs supported
   by normal reads to call somatic SVs - this is how minisv works with multiple
   samples jointly.

## <a name="call-sv"></a>Calling SVs

Minisv seamlessly works with SAM, PAF or GAF (graph PAF) formats. It
**requires** the `ds:Z` tag outputted by [minigraph][mg]-0.21+ or
[minimap2][mm2]-2.28+. For minigraph, use `-cxlr` for long reads. For minimap2,
use `-cxmap-hifi -s50 --ds` for HiFi reads. The minimap2 option for Nanopore
reads varies with the read error rate.

### <a name="call-germline"></a>Germline SVs

```sh
minisv.js extract -b data/hs38.cen-mask.bed aln.hg38l.paf.gz > sv.hg38l.rsv
cat sv.hg38l.rsv | sort -k1,1 -k2,2n -S4g | minisv.js merge - > sv.hg38l.msv
minisv.js genvcf sv.hg38l.msv > sv.hg38l.vcf
```
For calling germline SVs, you only need one linear reference genome. This is the
simplest use case. However, minisv does not infer genotypes. It is not the best
tool for germline SV calling.

### <a name="call-pair"></a>Somatic SVs in tumor-normal pairs

```sh
minisv.js e -n TUMOR -0b data/hs38.cen-mask.bed tumor.hg38l.paf tumor.chm13l.paf \
  tumor.chm13g.paf tumor-to-normal.self.paf | bash > sv-tumor.hg38l+tgs.rsv
minisv.js extract -n NORMAL normal.hg38l.paf > sv-normal.hg38l.rsv
cat sv-tumor.hg38l+tgs.rsv sv-normal.hg38l.rsv | sort -k1,1 -k2,2 -S4g \
  | minisv.js merge - | grep TUMOR | grep -v NORMAL > sv-paired.hg38l+tgs.msv
```
The last command selects SVs only present in TUMOR but not in NORMAL.

If you want to take the GRCh38 coordinate system, it is recommended to also
align reads to T2T-CHM13 and the [CHM13 graph][mg-zenodo]. If you have PacBio
HiFi reads for the normal, assemble the normal reads with [hifiasm][hifiasm],
align the tumor reads to the normal assembly and provide the alignment *as the
last input* along with option `-0`.

Graph alignment greatly reduces alignment errors when the normal assembly is
not available. When you have the normal assembly, intersecting with graph
alignment can be optional. While graph alignment still improves specificity, it
may affect sensitivity a little - a classical sensitivity vs specificity
problem.

Calling *de novo* SVs will be similar.

### <a name="call-tonly"></a>Large somatic SVs in tumor-only samples

```sh
minisv.js e -b data/hs38.cen-mask.bed tumor.hg38l.paf tumor.chm13l.paf \
  tumor.chm13g.paf | bash > sv.hg38l+tg.rsv
cat sv.hg38l+tg.rsv | sort -k1,1 -k2,2 -S4g | minisv.js merge - > sv.hg38l+tg.msv
minisv.js view -IC sv.hg38l+tg.msv
```
In the lack of the normal assembly in this case, pangenome graph alignment is
critical for reducing alignment errors and for filtering out common germline
SVs. There may be several hundred SVs called with this procedure. Most of the
called SVs below 10 kb are rare germline events, not somatic. Nevertheless,
most large chromosomal alterations, such as translocations, foldback
inversions (tagged by `foldback` in the output) and events longer 100 kb, are
likely somatic becase such large alterations rarely occur to germline.

### <a name="call-mosaic"></a>Mosaic SVs

```sh
minisv.js e -0b data/hs38.cen-mask.bed aln.hg38l.paf aln.chm13l.paf \
  aln.chm13g.paf aln.self.paf | bash > sv.hg38l+tgs.rsv
cat sv.hg38l+tgs.rsv | sort -k1,1 -k2,2 -S4g | minisv.js merge -c2 -s0 - > sv.hg38l+tgs.msv
```
Having the phased sample assembly is critical to the calling of small mosaic
SVs. If you do not have the assembly, please perform graph alignment. However,
because there are more small rare SVs than small mosaic SVs, only large mosaic
chromosomal alteration calls are reliable. If you have the assembly, graph
alignment may still help specificity but may hurt sensitivity around [VNTRs][vntr-wiki].

## <a name="compare"></a>Comparing SVs

The `eval` command of minisv compares two or multiple SV callsets. To compare
two callsets:
```sh
minisv.js eval -b data/hs38.reg.bed -l 100 call1.vcf call2.msv
```
where `-l` specifies the minimum SV length and `-b` specifies confident regions.
The command line outputs TP, FN and FP. Minisv considers two SVs, *S1* and
*S2*, to be the same if both ends of *S1* are within 500 bp from ends of *S2*
and the INDEL types of *S1* and *S2* are the same. Minisv compares all types of
SVs that can be associated with two ends. You can also specify the minimum read
support (`-c`) and the minimum SV length (`-l`)on the command line.

If more than two callsets are given on the command line, minisv will generate an
output like:
```txt
SN  980     0.6091  0.8580  0.6135  0.9104  0.8754  C1
SN  0.0673  110     0.0762  0.1319  0.1119  0.0665  C2
SN  0.8408  0.6727  958     0.6411  0.9216  0.8469  C3
SN  0.1980  0.4000  0.2119  326     0.2313  0.2154  C4
SN  0.7071  0.5818  0.7359  0.6074  536     0.7012  C5
SN  0.8459  0.5636  0.8361  0.6442  0.8731  947     C6
```
where the diagonal gives the count of SVs and the number at row *R* and column
*C* equals to the fraction of calls in *R* found in *C*. In this example,
84.59% of calls in C6 were found in C1 and 87.54% of calls in C1 found in C6.
Generally, higher fraction on a row is correlated with higher sensitivity;
higher fraction on a column is correlated with higher specificity for the
caller on the column.

Minisv seamlessly parses the VCF format and the minisv format. It has been
tested with Severus, Sniffles2, cuteSV, SAVANA, SVision-Pro, nanomonsv, SvABA
and GRIPSS.

## <a name="limit"></a>Limitations

1. Minisv simply counts supporting reads to call an SV. More complex
   algorithms, such as phasing, VNTR reposition and machine learning, may
   improve its accuracy.

2. Minisv does not output genotypes. This makes it less useful for germline SV
   calling. On the HG002 benchmark data, minisv is close to but not as good as
   the best germline SV caller.

[mg-zenodo]: https://zenodo.org/records/6286521
[vntr-wiki]: https://en.wikipedia.org/wiki/Variable_number_tandem_repeat
[mg]: https://github.com/lh3/minigraph
[mm2]: https://github.com/lh3/minimap2
[hifiasm]: https://github.com/chhylp123/hifiasm
