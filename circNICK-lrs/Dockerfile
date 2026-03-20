FROM mambaorg/micromamba:1.5.3

USER root

RUN apt-get update && apt-get install -y \
    git \
    wget \
    curl \
    build-essential \
    perl \
    && rm -rf /var/lib/apt/lists/*

RUN micromamba install -y -n base -c bioconda -c conda-forge \
    python=3.9 \
    bedtools=2.29.2 \
    samtools \
    pblat \
    nanofilt \
    deeptools \
    && micromamba clean -afy

ENV PATH=/opt/conda/bin:$PATH

RUN ln -s /opt/conda/bin/python3 /usr/local/bin/python3 && \
    ln -s /opt/conda/bin/python /usr/local/bin/python

RUN git clone https://github.com/omiics-dk/long_read_circRNA.git /opt/long_read_circRNA

RUN chmod +x /opt/long_read_circRNA/long_read_circRNA && \
    ln -s /opt/long_read_circRNA/long_read_circRNA /usr/local/bin/long_read_circRNA

RUN long_read_circRNA download-data \
    --installation-path /opt/long_read_circRNA \
    --data-output /opt/long_read_circRNA/data \
    --test-output /opt/long_read_circRNA/test_fastq

ENV SCRIPT_PATH=/opt/long_read_circRNA/scripts
ENV REFERENCE_PATH=/opt/long_read_circRNA/data

WORKDIR /data

ENTRYPOINT ["long_read_circRNA"]
CMD ["--help"]