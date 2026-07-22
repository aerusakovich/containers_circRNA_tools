FROM continuumio/miniconda3:23.5.2-0

LABEL authors="anrusakovich"
LABEL description="Docker image for NanoCirc's remap-based circRNA quantification module (minimap2 + samtools + pblat + pysam/pandas)"

# Ensure Python 3.11 is active
RUN conda install -y python=3.11 && conda clean -afy

# minimap2, samtools, pblat: the only external binaries the quantification
# scripts shell out to (confirmed by grepping the ported nanocirc_quant
# large_scale-track modules' subprocess calls — no bedtools dependency here)
RUN conda install -c bioconda -c conda-forge -y \
    minimap2 \
    samtools \
    pblat \
    && conda clean -afy

# pysam + pandas: the only two Python packages imported by the ported modules
# (locus_dedup, circle_refs, genome_align, rescue_pool, chunked_remap,
# tool_readset_union, coverage_flagging, cross_locus_similarity, tiered_rescue)
RUN pip install --no-cache-dir pysam pandas

WORKDIR /data
