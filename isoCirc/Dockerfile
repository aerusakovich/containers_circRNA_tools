FROM ubuntu:22.04

LABEL authors="anrusakovich"
LABEL tool="isoCirc"
LABEL description="isoCirc 1.0.7 + dependencies (Ubuntu 22.04, x86_64/AMD64)"

ENV LC_ALL=C
ENV LANG=C
ENV DEBIAN_FRONTEND=noninteractive
ENV PATH="/usr/local/bin:/usr/bin:/bin:$PATH"
ENV PIP_NO_CACHE_DIR=1

RUN apt-get update && apt-get install -y --no-install-recommends \
    ca-certificates curl wget git \
    build-essential pkg-config make \
    python3 python3-pip python3-dev \
    zlib1g-dev libbz2-dev liblzma-dev libcurl4-gnutls-dev libssl-dev \
    libsqlite3-0 \
    minimap2 bedtools gawk \
    python3-setuptools python3-wheel python3-distutils && \
    update-alternatives --install /usr/bin/awk awk /usr/bin/gawk 50 && \
    ln -sf /usr/bin/gawk /usr/bin/awk && \
    apt-mark manual gawk

RUN python3 -m pip install --upgrade pip && \
    python3 -m pip install "isocirc==1.0.7"

RUN git clone --depth 1 https://github.com/Xinglab/isoCirc.git /opt/isoCirc && \
    ln -s /opt/isoCirc /isoCirc

RUN cd /usr/local/bin && \
    wget -q https://hgdownload.soe.ucsc.edu/admin/exe/linux.x86_64/gtfToGenePred -O gtfToGenePred && \
    wget -q https://hgdownload.soe.ucsc.edu/admin/exe/linux.x86_64/genePredToBed -O genePredToBed && \
    chmod +x gtfToGenePred genePredToBed

RUN rm -rf /var/lib/apt/lists/* /root/.cache/pip

ENTRYPOINT ["isocirc"]