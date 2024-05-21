## Centromere regions

File `chm13v2.cen-mask.bed` gives the approximate locations of centromeric
satellite repeats and acrocentric short arms in CHM13 v2.0. It was *manually*
constructed based on the [official satellite][cen-sat] annotation, the
[DNA-BRNN][dna-brnn] satellite annotation and the [minigraph pangenome
graph][HPRC-mg] from the HPRC year-1 data. Alignment in the BED regions is
usually much worse than in the rest of the genome. This file is intended for
filtering spurious alignments or variant calls caused by centromeric repeats.

File `hs38.cen-mask.bed` was constructed similarly from DNA-BRNN annotation and
regions where minigraph alignment faded.

These files are also available [on Zenodo][zenodo].

[cen-sat]: https://s3-us-west-2.amazonaws.com/human-pangenomics/T2T/CHM13/assemblies/annotation/chm13v2.0_censat_v2.0.bed
[dna-brnn]: https://github.com/lh3/dna-brnn
[HPRC-mg]: https://zenodo.org/records/10693675
[zenodo]: https://zenodo.org/records/10963019
