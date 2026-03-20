FROM mambaorg/micromamba:1.5.3

USER root

# Install system dependencies + perl
RUN apt-get update && apt-get install -y \
    git \
    wget \
    curl \
    build-essential \
    perl \
    && rm -rf /var/lib/apt/lists/*

# Install all dependencies into base environment using mamba
RUN micromamba install -y -n base -c bioconda -c conda-forge \
    python=3.9 \
    bedtools=2.29.2 \
    samtools \
    pblat \
    nanofilt \
    deeptools \
    && micromamba clean -afy

# Make conda/mamba binaries available system-wide
ENV PATH=/opt/conda/bin:$PATH

# Symlink python3 so scripts using #!/usr/bin/env python3 can find it
RUN ln -s /opt/conda/bin/python3 /usr/local/bin/python3 && \
    ln -s /opt/conda/bin/python /usr/local/bin/python

# Clone the repository
RUN git clone https://github.com/omiics-dk/long_read_circRNA.git /opt/long_read_circRNA

# Make executable and available system-wide
RUN chmod +x /opt/long_read_circRNA/long_read_circRNA && \
    ln -s /opt/long_read_circRNA/long_read_circRNA /usr/local/bin/long_read_circRNA

# Download required reference data
RUN long_read_circRNA download-data \
    --data-output /opt/long_read_circRNA/data \
    --test-output /opt/long_read_circRNA/test_fastq

ENV SCRIPT_PATH=/opt/long_read_circRNA/scripts
ENV REFERENCE_PATH=/opt/long_read_circRNA/data

WORKDIR /data

ENTRYPOINT ["long_read_circRNA"]
CMD ["--help"]