FROM continuumio/miniconda3:latest

LABEL description="isoCirc - circular RNA isoform analysis tool"
LABEL url="https://github.com/Xinglab/isoCirc"

# Install system dependencies
RUN apt-get update && apt-get install -y \
    bzip2 \
    git \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

# Install bedtools and minimap2 via conda
RUN conda install -c bioconda -c conda-forge -y \
    bedtools \
    minimap2 \
    && conda clean -afy

# Install isoCirc via pip
RUN pip install --no-cache-dir isocirc

# Clone the source repo to get the pipeline scripts and bundled bin deps
RUN git clone https://github.com/Xinglab/isoCirc.git /opt/isoCirc

# Copy the bundled bin dependencies into PATH
RUN cp /opt/isoCirc/isoCirc_pipeline/bin/* /usr/local/bin/ \
    && chmod +x /usr/local/bin/*

# Create a wrapper so `isocirc` works as a plain command
RUN printf '#!/bin/bash\nexec python /opt/isoCirc/isoCirc_pipeline/isocirc/isocirc "$@"\n' \
    > /usr/local/bin/isocirc \
    && chmod +x /usr/local/bin/isocirc

# Set the working directory
WORKDIR /data

CMD ["isocirc", "--help"]