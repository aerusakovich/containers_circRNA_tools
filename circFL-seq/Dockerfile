# syntax=docker/dockerfile:1
FROM condaforge/mambaforge:latest
LABEL authors="anrusakovich"
LABEL description="Docker image for circFL-seq"

ENV DEBIAN_FRONTEND=noninteractive
ENV LC_ALL=C
ENV PYTHONUNBUFFERED=1

RUN apt-get update && apt-get install -y libgl1 libglib2.0-0 wget && \
    apt-get clean && rm -rf /var/lib/apt/lists/*

RUN mamba create -y -n circFL -c conda-forge -c bioconda \
        python=3.9 samtools bcftools minimap2 bedtools trf \
        tidehunter pysam numpy pandas portion \
        scikit-learn pyfasta mappy progressbar docopt porechop && \
    /opt/conda/envs/circFL/bin/pip install circfull && \
    chmod +x /opt/conda/envs/circFL/bin/* && \
    conda clean -afy

RUN sed -i 's/filterOut(RG_outPrefix,RG_outPrefix,thread,rmskFile)/filterOut(RG_outPrefix,RG_outPrefix,fastq,genome,thread,rmskFile)/' \
    /opt/conda/envs/circFL/lib/python3.9/site-packages/circfull/RG.py

ENV PATH="/opt/conda/envs/circFL/bin:/opt/conda/bin:$PATH"
ENTRYPOINT ["/opt/conda/envs/circFL/bin/circfull"]