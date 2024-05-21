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
- [Design](#design)
- [Calling SVs](#call-sv)
  - [Germline SVs](#call-germline)
  - [Somatic SVs in tumor-normal pairs](#call-pair)
  - [Large somatic SVs from a tumor-only sample](#call-tonly)
  - [Mosaic SVs](#call-mosaic)
- [Comparing SVs](#compare)

## <a name="intro"></a>Introduction

Minisv is a lightweight structural variation (SV) caller for long genomic
reads. It is primarily developed for calling mosaic SVs, somatic SVs in paired
or unpaired tumor samples, or de novo SVs. While minisv can also call germline
SVs, it does not output genotypes and is a little less accurate than the best
germline SV callers.

The key advantage of minisv is to combine read alignments against multiple
references, including pangenome graphs and the assembly of input reads. Minisv
calls a SV only if it can be observed in all alignment. This simple strategy
reduces alignment errors and filters out germline SVs in the sample (when used
with the assembly of input reads) or in the population (when used with
pangenomes).

For PacBio HiFi reads at high coverage, minisv achieves much higher specificity
for mosaic SV calling, has comparable accuracy to tumor-normal paired SV
callers, and is the only tool accurate enough for calling large chromosomal
alterations from a single tumor sample without matched normal.

## <a name="design"></a>Design

Minisv can call 1) germline SVs, 2) somatic SVs in tumor-normal pairs, 3) large
somatic SVs in a single tumor, 4) mosaic SVs and 5) de novo SVs in a trio. The
exact use also depends on whether the de novo assembly of the sample is
available. Minisv achieves this variety of calling modes in three steps:

1. Extract SVs from read alignment with `extract`. This command processes one
   read at a time, extracts long INDELs or breakends and outputs in a BED-like
   minisv format. If you have tumor-normal pairs, remember to use `-n` to
   specify samples such thatdd you can distinguish the samples in step 3.

2. Intersect candidate SVs extracted from multiple alignment files with `isec`.
   You can skip this step if you use one reference genome only, but you would
   lose the key advantage of minisv. The first two steps are shared by all
   calling modes. You can combine them together with the `e` command, which
   provides a more convenient but less flexible interface.

3. Call SVs supported by multiple reads using `merge`. This command counts the
   number of supporting reads for each sample. You can filter out SVs supported
   by normal reads to call somatic SVs - this is how minisv works with multiple
   samples jointly.

## <a name="call-sv"></a>Calling SVs

Minisv seamlessly works with SAM, PAF or GAF (graph PAF) formats. It
**requires** the `ds:Z` tag outputted by [minigraph][mg]-0.21+ or
[minimap2][mm2]-2.28+. For minigraph, use `-cxlr` for long reads. For minimap2,
use `-cxmap-hifi -s50` for HiFi reads. The minimap2 option for Nanopore reads
varies with the read error rate.

### <a name="call-germline"></a>Germline SVs

```sh
minisv.js extract -b data/hs38.cen-mask.bed aln.hg38l.paf.gz > sv.hg38l.rsv
cat sv.hg38l.rsv | sort -k1,1 -k2,2n -S4g | minisv.js merge - > sv.hg38l.msv
minisv.js genvcf sv.hg38l.msv > sv.hg38l.vcf
```

For calling germline SVs, you only need one linear reference genome. This is the
simplest use case, but as is mentioned above, minisv is not optimized for
germline SV calling.

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
HiFi reads for the normal, assemble the reads with hifiasm, align the tumor
reads to the normal assembly and provide the alignment *as the last input*
along with option `-0`.

When you do not have the normal assembly, intersecting with the graph alignment
greatly reduces alignment errors at a minor cost of sensitivity. The
availability of the normal assembly is usually more effective than graph
alignment. When you have the normal assembly and worry about sensitivity, you
may drop the graph alignment.

Calling *de novo* SVs will be similar.

### <a name="call-tonly"></a>Large somatic SVs from a tumor-only sample

```sh
minisv.js e -b data/hs38.cen-mask.bed tumor.hg38l.paf tumor.chm13l.paf \
  tumor.chm13g.paf | bash > sv.hg38l+tg.rsv
cat sv.hg38l+tg.rsv | sort -k1,1 -k2,2 -S4g | minisv.js merge - > sv.hg38l+tg.msv
minisv.js view -IC sv.hg38l+tg.msv
```
In this example, pangenome graph alignment is critical for reducing alignment
errors and for filtering out common germline SVs. There may be several hundred
SVs called with this procedure. Most of SVs below 10 kb are rare germline SVs.
Most large chromosomal alterations, such as translocations, foldback inversions
(tagged by `foldback` in the output) and SVs longer 100 kb, are very rare in
germline and are thus likely tumor events.

Note that other SV callers may call ~30 large chromosomal alterations from a
normal sample. Minisv has a much lower background error rate. Calling large
chromosomal alterations from a tumor-only sample is a unique advantage of
minisv.

### <a name="call-mosaic"></a>Mosaic SVs

```sh
minisv.js e -0b data/hs38.cen-mask.bed aln.hg38l.paf aln.chm13l.paf \
  aln.chm13g.paf aln.self.paf | bash > sv.hg38l+tgs.rsv
cat sv.hg38l+tgs.rsv | sort -k1,1 -k2,2 -S4g | minisv.js merge -c2 -s0 - > sv.hg38l+tgs.msv
```
Having the phased sample assembly is **critical** to the calling of small
mosaic SVs. If you do not have the assembly, graph alignment is important for
reducing alignment errors and filtering common germline SVs. However, because
there are more small rare SVs than small mosaic SVs, only large mosaic
chromosomal alteration calls are reliable. If you have the assembly, graph
alignment may still help specificity at the cost of sensitivity especially
around [VNTRs][vntr-wiki].

## <a name="compare"></a>Comparing SVs

[mg-zenodo]: https://zenodo.org/records/6286521
[vntr-wiki]: https://en.wikipedia.org/wiki/Variable_number_tandem_repeat
[mg]: https://github.com/lh3/minigraph
[mm2]: https://github.com/lh3/minimap2
