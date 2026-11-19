# snpArcher - UCL Speciation Group

The snpArcher pipeline modified for use in pixy-style computation of statistics (i.e. explicitly accounting for callable invariant sites) and ARG inference. Based on snpArcher v2.3.0. 

### GATK modifications

#### (1) HaplotypeCaller

Modified in `workflow/rules/variant_calling/gatk_intervals.smk` (rule `gatk_haplotypecaller_interval`, used when `intervals: enabled: true`) and `workflow/rules/variant_calling/gatk.smk` (rule `gatk_haplotypecaller`, used otherwise). (Note that **comments after \ break the command** - included here to highlight modifications.)

```
# For the --gvcf-gq-bands flag; see below.
params:
    gq_bands=" ".join(f"--gvcf-gq-bands {i}" for i in range(1,100)),

# The command from gatk.smk (the interval version also has -L {input.interval})
gatk HaplotypeCaller \
    --java-options '-Xmx{resources.mem_mb_reduced}m' \
    -R {input.ref} \
    -I {input.bam} \
    --read-index {input.bam_index} \
    -O {output.gvcf} \
    -ploidy {params.ploidy} \
    --native-pair-hmm-threads {threads} \
    --emit-ref-confidence GVCF \
    --min-pruning {params.min_pruning} \
    --min-dangling-branch-length {params.min_dangling} \
    --do-not-run-physical-phasing true \ # MODIFIED - the PS tags would be removed before actual phasing anyway
    {params.gq_bands} \ # MODIFIED - This is to ameliorate a bias in DP, since what's recorded as the FORMAT/DP tags is the per-block minimum. Probably a negligible effect. 
    &> {log}
```

#### (2) GenomicsDBImport

Left at snpArcher's settings.

#### (3) GenotypeGVCFs

Modified in `workflow/rules/variant_calling/joint_gvcf_intervals.smk` (rule `gatk_genotype_gvcfs`, used when `intervals: enabled: true`) and `workflow/rules/variant_calling/joint_gvcf.smk` (rule `joint_genotype_gvcfs`, used otherwise). (Note that **comments after \ break the command** - included here to highlight modifications.)

```
# The command (identical in both rules)
gatk GenotypeGVCFs \
    --java-options '-Xmx{resources.mem_mb_reduced}m' \
    -R {input.ref} \
    --heterozygosity {params.het_prior} \
    --genomicsdb-shared-posixfs-optimizations true \
    -V gendb://"$EXTRACT_DIR/{params.db_rel}" \
    -O {output.vcf} \
    --tmp-dir {resources.tmpdir} \
    --include-non-variant-sites true \ # MODIFIED - ESSENTIAL for all sites output
    &> {log}
```

Another change is in `workflow_profile/config.yaml`. To unpack the archives created by GenomicsDBImport, GenotypeGVCFs uses `{resources.tmpdir}`. By default, `tmpdir` is `system_tmpdir`, i.e. the compute node's `/tmp`, which appears to be capped at ~5.4 GB per job.

(You will want to include a default `config.yaml` for our cluster.)

```
set-resources:
  gatk_genotype_gvcfs:      # used when intervals: enabled: true
    ...
    tmpdir: "'/SAN/ugi/littorina_shortread/Tymek/LitRepro/calls_2/tmp_genotype'"
  joint_genotype_gvcfs:     # used when intervals are disabled
    ...
    tmpdir: "'/SAN/ugi/littorina_shortread/Tymek/LitRepro/calls_2/tmp_genotype'"
```

#### Claude-implemented, not checked

**GenotypeGVCFs on multi-contig shards** - `workflow/rules/variant_calling/joint_gvcf_intervals.smk`, rule `gatk_genotype_gvcfs` (only the intervals version; `joint_gvcf.smk` is unchanged).

Problem: shards bundling >50 unplaced scaffolds are imported with `--merge-contigs-into-num-partitions` (set by snpArcher). With `--include-non-variant-sites true` and no `-L`, GenotypeGVCFs failed on all of these (41/513 shards) with `GenomicsDBException: Contig NC_090245.1 not found, or found multiple times in vid` - apparently it walks the reference from its first contig, which is not in the merged workspace. Shards with 1-4 contigs were unaffected, and in the earlier run (before `--include-non-variant-sites` was added) shards with >50 contigs genotyped fine.

Fix: give GenotypeGVCFs the shard's interval list, merged the same way as in GenomicsDBImport.

```
input:
    db="results/gatk_genomics_db/L{interval}.tar",
    interval="results/intervals/db/{interval}-scattered.interval_list", # MODIFIED
    ...

gatk GenotypeGVCFs \
    ...
    -V gendb://"$EXTRACT_DIR/{params.db_rel}" \
    -L {input.interval} \ # MODIFIED
    --merge-input-intervals \ # MODIFIED - as in GenomicsDBImport, so the output covers the same regions
    ...
```

In `calls_2`, only the 41 failed shards were rerun with this (the other 472 were genotyped without `-L`); all 41 then completed.


#### For the future

- QC dashboard will be difficult to generate based on an all-sites vcf - this should use a temporary variants-only intermediate
- Ask yourself what other modules would need adjustments to account for all-sites VCF
