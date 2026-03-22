FROM mambaorg/micromamba:1.5.3

USER root

RUN apt-get update && apt-get install -y \
    git \
    wget \
    curl \
    build-essential \
    && rm -rf /var/lib/apt/lists/*

RUN micromamba install -y -n base -c bioconda -c conda-forge -c defaults \
    python=3.7.6 \
    numpy=1.21.2 \
    bedtools=2.29.2 \
    samtools=1.9 \
    pblat=2.5 \
    nanofilt=2.8.0 \
    deeptools=3.5.1 \
    deeptoolsintervals=0.1.9 \
    pybigwig=0.3.17 \
    py2bit=0.3.0 \
    pysam=0.15.3 \
    biopython=1.78 \
    pandas=1.3.5 \
    matplotlib-base=3.4.2 \
    scipy=1.7.3 \
    perl=5.26.2 \
    htslib=1.9 \
    && micromamba clean -afy

ENV PATH=/opt/conda/bin:$PATH

RUN ln -s /opt/conda/bin/python3 /usr/local/bin/python3 && \
    ln -s /opt/conda/bin/python /usr/local/bin/python

RUN git clone https://github.com/omiics-dk/long_read_circRNA.git /opt/long_read_circRNA

# Fix naming mismatch: scripts expect *.scan.circRNA.sort.bam but tool produces *.circRNA.sort.bam
RUN sed -i 's/\$sample\.scan\.circRNA\.sort\.bam/\$sample.circRNA.sort.bam/g' \
        /opt/long_read_circRNA/scripts/extra_stuff_v2.sh \
        /opt/long_read_circRNA/scripts/novel_exons_and_alternative_usage_v7.0.sh

RUN chmod +x /opt/long_read_circRNA/long_read_circRNA && \
    ln -s /opt/long_read_circRNA/long_read_circRNA /usr/local/bin/long_read_circRNA

RUN long_read_circRNA download-data \
    --installation-path /opt/long_read_circRNA \
    --data-output /opt/long_read_circRNA/data \
    --test-output /opt/long_read_circRNA/test_fastq

RUN find /opt/long_read_circRNA/data -name "*.fa" -o -name "*.fasta" | \
    xargs -I {} samtools faidx {}

ENV SCRIPT_PATH=/opt/long_read_circRNA/scripts
ENV REFERENCE_PATH=/opt/long_read_circRNA/data

WORKDIR /data

ENTRYPOINT ["long_read_circRNA"]
CMD ["--help"]