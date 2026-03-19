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
    bedtools>=2.27.0 \
    minimap2>=2.11 \
    && conda clean -afy

# Install isoCirc via pip
RUN pip install --no-cache-dir isocirc

# Clone the source repo to get the pipeline bin scripts (required for full functionality)
RUN git clone https://github.com/Xinglab/isoCirc.git /opt/isoCirc

# Copy the bundled bin dependencies into PATH (as required by the cluster workaround)
RUN cp /opt/isoCirc/isoCirc_pipeline/bin/* /usr/local/bin/ \
    && chmod +x /usr/local/bin/*

# Set the working directory
WORKDIR /data

# Expose the correct entrypoint — direct python call, since bare `isocirc` may not work
ENTRYPOINT ["python", "/opt/isoCirc/isoCirc_pipeline/isocirc/isocirc"]
CMD ["--help"]