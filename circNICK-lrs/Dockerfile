FROM continuumio/miniconda3:latest

# Install system dependencies
RUN apt-get update && apt-get install -y \
    git \
    wget \
    curl \
    build-essential \
    && rm -rf /var/lib/apt/lists/*

# Clone the repository
RUN git clone https://github.com/omiics-dk/long_read_circRNA.git /opt/long_read_circRNA

# Create and configure conda environment
WORKDIR /opt/long_read_circRNA
RUN conda env create -f long_read_circRNA.yml && \
    conda clean -afy

# Make the tool executable
RUN chmod +x /opt/long_read_circRNA/long_read_circRNA

# Add tool to PATH inside the conda environment
SHELL ["conda", "run", "-n", "long_read_circRNA", "/bin/bash", "-c"]

# Download required reference data into the container
RUN /opt/long_read_circRNA/long_read_circRNA download-data \
    --reference-path /opt/long_read_circRNA/data

# Set default script path via environment variable
ENV SCRIPT_PATH=/opt/long_read_circRNA/scripts
ENV REFERENCE_PATH=/opt/long_read_circRNA/data

# Ensure conda env is activated on container start
RUN echo "conda activate long_read_circRNA" >> ~/.bashrc
ENV PATH=/opt/conda/envs/long_read_circRNA/bin:$PATH

WORKDIR /data

ENTRYPOINT ["conda", "run", "--no-capture-output", "-n", "long_read_circRNA", \
            "/opt/long_read_circRNA/long_read_circRNA"]
CMD ["--help"]