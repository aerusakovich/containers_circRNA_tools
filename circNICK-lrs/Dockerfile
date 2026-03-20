FROM continuumio/miniconda3:latest

# Install system dependencies + perl
RUN apt-get update && apt-get install -y \
    git \
    wget \
    curl \
    build-essential \
    perl \
    && rm -rf /var/lib/apt/lists/*

# Install all dependencies directly into the base conda environment
RUN conda install -y -c bioconda -c conda-forge \
    bedtools=2.29.2 \
    samtools \
    pblat \
    nanofilt \
    deeptools \
    && conda clean -afy

# Clone the repository
RUN git clone https://github.com/omiics-dk/long_read_circRNA.git /opt/long_read_circRNA

# Make executable and available system-wide
RUN chmod +x /opt/long_read_circRNA/long_read_circRNA && \
    ln -s /opt/long_read_circRNA/long_read_circRNA /usr/local/bin/long_read_circRNA

# Download required reference data
RUN long_read_circRNA download-data \
    --data-output /opt/long_read_circRNA/data \
    --test-output /opt/long_read_circRNA/test_fastq

# Pre-set script and reference paths
ENV SCRIPT_PATH=/opt/long_read_circRNA/scripts
ENV REFERENCE_PATH=/opt/long_read_circRNA/data

WORKDIR /data

ENTRYPOINT ["long_read_circRNA"]
CMD ["--help"]