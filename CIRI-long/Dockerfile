FROM python:3.8

RUN apt-get update && apt-get install -y \
    git \
    make \
    python3 \
    python3-pip \
    build-essential \
    && rm -rf /var/lib/apt/lists/*

RUN pip3 install CIRI-long==1.1.0

ENV PATH=/usr/local/bin:$PATH

ENTRYPOINT ["ciri-long"]