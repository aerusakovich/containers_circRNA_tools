# syntax=docker/dockerfile:1
FROM condaforge/mambaforge:latest

LABEL authors="anrusakovich"
LABEL description="Docker image for circFL-seq"

ENV DEBIAN_FRONTEND=noninteractive
ENV LC_ALL=C
ENV PYTHON_UNBUFFERED=1

RUN mamba create -y -n circFL -c conda-forge -c bioconda \
        python=3.9 samtools bcftools minimap2 bedtools trf \
        tidehunter pysam portion \
        scikit-learn pyfasta mappy progressbar docopt porechop && \
    /opt/conda/envs/circFL/bin/pip install circfull && \
    /opt/conda/envs/circFL/bin/pip install \
        "pandas==1.5.3" \
        "numpy==1.24.4" \
        --force-reinstall && \
    chmod +x /opt/conda/envs/circFL/bin/* && \
    conda clean -afy

RUN PKG_DIR="/opt/conda/envs/circFL/lib/python3.9/site-packages/circfull" && \
    sed -i "s/filterOut(RG_outPrefix,RG_outPrefix,thread,rmskFile)/filterOut(RG_outPrefix,RG_outPrefix,fastq,genome,thread,rmskFile)/" \
        "${PKG_DIR}/RG.py" && \
    sed -i "s/set(iso2read\.keys())/list(iso2read.keys())/g" \
        "${PKG_DIR}/mRG_merge.py" && \
    sed -i "s/\['intergenic','antisense','read through'\]/['intergenic','antisense','read through','unannotated']/g" \
        "${PKG_DIR}/annoFL.py"

RUN <<'PYEOF'
#!/usr/bin/env python3
import sys

PKG = "/opt/conda/envs/circFL/lib/python3.9/site-packages/circfull"

with open(f"{PKG}/DNSC_getNovo.py") as f:
    lines = f.readlines()

patched = []
for line in lines:
    patched.append(line)
    if "thFile=pd.read_csv" in line:
        indent = "    "
        patched.append(
            indent
            + "if thFile.empty: "
            + "print('#### Warning: DNSC - TideHunter.tab is empty ####', file=sys.stderr); "
            + "thFile.to_csv(outPrefix+'TideHunter_Pass.tab', sep='\\t', index=False); "
            + "return\n"
        )

with open(f"{PKG}/DNSC_getNovo.py", "w") as f:
    f.writelines(patched)

print("DNSC_getNovo.py patched OK", file=sys.stderr)

with open(f"{PKG}/annoFL.py") as f:
    lines = f.readlines()

patched = []
i = 0
while i < len(lines):
    line = lines[i]

    if "if tmp['chr'] not in tabixfile.contigs:" in line:
        stripped = line.lstrip()
        indent = line[: len(line) - len(stripped)]
        bi = indent + "    "
        patched.append(line)
        patched.append(bi + "start_type=','.join(['unannotated']* len(exon_start))\n")
        patched.append(bi + "end_type=','.join(['unannotated']* len(exon_end))\n")
        patched.append(bi + "circ_type='unannotated'\n")
        patched.append(bi + "geneName=''\n")
        patched.append(bi + "return(start_type,end_type,circ_type,geneName)\n")
        i += 1
        continue

    if "for gtf in tabixfile.fetch(" in line and "pysam.asGTF()" in line:
        stripped = line.lstrip()
        indent = line[: len(line) - len(stripped)]
        bi = indent + "    "
        patched.append(indent + "try:\n")
        patched.append(bi + "it = tabixfile.fetch(tmp['chr'], tmp['start']-1, tmp['end']-1,parser=pysam.asGTF())\n")
        patched.append(indent + "except ValueError:\n")
        patched.append(bi + "start_type=','.join(['unannotated']* len(exon_start))\n")
        patched.append(bi + "end_type=','.join(['unannotated']* len(exon_end))\n")
        patched.append(bi + "circ_type='unannotated'\n")
        patched.append(bi + "geneName=''\n")
        patched.append(bi + "return(start_type,end_type,circ_type,geneName)\n")
        patched.append(indent + "for gtf in it:\n")
        i += 1
        continue

    if "elif i=='intergenic':" in line:
        stripped = line.lstrip()
        indent = line[: len(line) - len(stripped)]
        bi = indent + "    "
        patched.append(line)
        i += 1
        if i < len(lines):
            patched.append(lines[i]); i += 1
        if i < len(lines):
            patched.append(lines[i]); i += 1
        patched.append(indent + "elif i=='unannotated':\n")
        patched.append(bi    + "n='unannotated'\n")
        patched.append(bi    + "m='unannotated'\n")
        continue

    patched.append(line)
    i += 1

with open(f"{PKG}/annoFL.py", "w") as f:
    f.writelines(patched)

print("annoFL.py patched OK", file=sys.stderr)
PYEOF

ENV PATH="/opt/conda/envs/circFL/bin:/opt/conda/bin:$PATH"

ENTRYPOINT ["/opt/conda/envs/circFL/bin/circfull"]