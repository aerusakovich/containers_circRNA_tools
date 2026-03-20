FROM continuumio/miniconda3:23.5.2-0

LABEL description="isoCirc - circular RNA isoform analysis tool"
LABEL url="https://github.com/Xinglab/isoCirc"

# Install system dependencies including C build tools (required for mappy and crlibm)
RUN apt-get update && apt-get install -y \
    bzip2 \
    git \
    gcc \
    make \
    build-essential \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

# Ensure Python 3.11 is active
RUN conda install -y python=3.11 && conda clean -afy

# Install bedtools >= 2.27.0 and minimap2 >= 2.11 via conda (pinned to satisfy isoCirc deps)
RUN conda install -c bioconda -c conda-forge -y \
    "bedtools>=2.27.0" \
    "minimap2>=2.11" \
    && conda clean -afy

# Install isoCirc via pip
RUN pip install --no-cache-dir isocirc

# Clone the source repo to get the pipeline scripts and bundled bin deps
RUN git clone https://github.com/Xinglab/isoCirc.git /opt/isoCirc

# Copy the bundled bin dependencies into PATH
RUN cp /opt/isoCirc/isoCirc_pipeline/bin/* /usr/local/bin/ \
    && chmod +x /usr/local/bin/*

# Add a wrapper so `isocirc` works as a plain command (required for singularity exec)
RUN printf '#!/bin/bash\npython /opt/isoCirc/isoCirc_pipeline/isocirc/isocirc "$@"\n' \
    > /usr/local/bin/isocirc \
    && chmod +x /usr/local/bin/isocirc

WORKDIR /data

ENTRYPOINT ["isocirc"]
CMD ["--help"]