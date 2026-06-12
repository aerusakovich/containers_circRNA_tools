FROM continuumio/miniconda3:23.5.2-0

LABEL description="isoCirc - circular RNA isoform analysis tool"
LABEL url="https://github.com/Xinglab/isoCirc"

# Install system dependencies including C build tools (required for mappy and crlibm)
# gawk is required by bundled itst_gtf_gtf script — mawk (debian default) lacks GNU awk features
RUN apt-get update && apt-get install -y \
    bzip2 \
    git \
    gcc \
    make \
    build-essential \
    gawk \
    && update-alternatives --install /usr/bin/awk awk /usr/bin/gawk 100 \
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

# Fix BioPython >=1.80 incompatibility in get_cigar_from_pairwise_res (parse_bam.py).
# v1.0.7 requires biopython>=1.80 but left the text-format parser using old indices:
#   pre-1.80 middle line: "0 |||...||| 19"  → rsplit ele[5]=aln_str, ele[9]=query_seq
#   1.80+    middle line: "|||...||| "      → rsplit ele[4]=aln_str, ele[7]=query_seq
#                         (position numbers removed; Score: line appended)
# Detect by checking ele[4]: if it is all |/./- chars → new format, else old format.
RUN python3 - << 'PATCH'
import isocirc.parse_bam as pb
src = pb.__file__
with open(src) as f:
    code = f.read()
old = "target_str, query_str, aln_str = ele[2], ele[9], ele[5]"
new = (
    "target_str = ele[2]\n"
    "    if len(ele) >= 5 and all(c in '|.-' for c in ele[4]):\n"
    "        aln_str, query_str = ele[4], ele[7]  # BioPython >=1.80: no pos nums on indicator line\n"
    "    else:\n"
    "        aln_str, query_str = ele[5], ele[9]  # pre-1.80: indicator line had '0 str end'"
)
assert old in code, "Pattern not found — parse_bam.py may have changed: " + src
with open(src, "w") as f:
    f.write(code.replace(old, new))
print("Patched", src)
PATCH

# Add a wrapper so `isocirc` works as a plain command (required for singularity exec)
RUN printf '#!/bin/bash\npython /opt/isoCirc/isoCirc_pipeline/isocirc/isocirc "$@"\n' \
    > /usr/local/bin/isocirc \
    && chmod +x /usr/local/bin/isocirc

WORKDIR /data

ENTRYPOINT ["isocirc"]
CMD ["--help"]