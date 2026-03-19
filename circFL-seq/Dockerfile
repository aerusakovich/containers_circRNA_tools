# syntax=docker/dockerfile:1
FROM condaforge/mambaforge:latest
LABEL authors="anrusakovich"
LABEL description="Docker image for circFL-seq (patched)"

ENV DEBIAN_FRONTEND=noninteractive
ENV LC_ALL=C
ENV PYTHONUNBUFFERED=1
ENV PKG=/opt/conda/envs/circFL/lib/python3.9/site-packages/circfull

RUN apt-get update && apt-get install -y libgl1 libglib2.0-0 wget && \
    apt-get clean && rm -rf /var/lib/apt/lists/*

RUN mamba create -y -n circFL -c conda-forge -c bioconda \
        python=3.9 samtools bcftools minimap2 bedtools trf \
        tidehunter pysam numpy pandas portion \
        scikit-learn pyfasta mappy progressbar docopt porechop && \
    /opt/conda/envs/circFL/bin/pip install circfull && \
    chmod +x /opt/conda/envs/circFL/bin/* && \
    conda clean -afy

# ── Patch 1: RG_constructFL.py ────────────────────────────────────────────────
# Insert an early-return guard when FL_result is empty, before the
# FL_result['len'] = FL_result.apply(getExonLen ...) line which crashes
# on an empty DataFrame.
RUN python3 - <<'EOF'
import sys

path = '/opt/conda/envs/circFL/lib/python3.9/site-packages/circfull/RG_constructFL.py'
with open(path) as f:
    lines = f.readlines()

patched = False
new_lines = []
i = 0
while i < len(lines):
    line = lines[i]
    if not patched and "FL_result['len']=FL_result.apply(getExonLen" in line.replace(" ", ""):
        indent = line[:len(line) - len(line.lstrip())]
        new_lines.append(indent + "if FL_result.empty:\n")
        new_lines.append(indent + "    import sys; print('cRG constructFL: no valid FL-circRNAs, skipping', file=sys.stderr)\n")
        new_lines.append(indent + "    return\n")
        patched = True
    new_lines.append(line)
    i += 1

if not patched:
    print("ERROR: RG_constructFL.py target line not found", file=sys.stderr)
    sys.exit(1)

with open(path, 'w') as f:
    f.writelines(new_lines)
print("Patched RG_constructFL.py OK")
EOF

# ── Patch 2: RG.py ────────────────────────────────────────────────────────────
# Three fixes consolidated:
#   a) Add missing 'fastq' and 'genome' args to filterOut() call (original fix).
#   b) Guard adjFL / circSeq / annotateNormal behind a check for
#      constructFL_Normal.txt so the pipeline continues when that file is absent.
#   c) Guard circFL_output behind a check for result_Normal.txt for the same reason.
RUN python3 - <<'EOF'
import sys

path = '/opt/conda/envs/circFL/lib/python3.9/site-packages/circfull/RG.py'
with open(path) as f:
    src = f.read()

# ---- a) filterOut argument fix ----
old_filter = 'filterOut(RG_outPrefix,RG_outPrefix,thread,rmskFile)'
new_filter = 'filterOut(RG_outPrefix,RG_outPrefix,fastq,genome,thread,rmskFile)'
if old_filter in src:
    src = src.replace(old_filter, new_filter)
    print("  OK  : filterOut() argument fix")
else:
    print("  WARN: filterOut() target not found — already patched or signature changed", file=sys.stderr)

# ---- b) adjFL / circSeq / annotateNormal guard ----
old_block = (
    "    plog('Adjust FL-circRNA: adjFL')\n"
    "    adjFL([genome,anno,RG_outPrefix,thread])\n"
    "    plog('Adjust mistaken circRNA: circSeq')\n"
    "    circSeq(genome,RG_outPrefix,thread)\n"
    "    \n"
    "    plog('Annotate circRNA: annotateNormal')\n"
    "    annotateNormal([RG_outPrefix,anno])\n"
)
new_block = (
    "    if not os.path.exists(RG_outPrefix+'constructFL_Normal.txt'):\n"
    "        import sys; print('Skipping adjFL/circSeq/annotateNormal: no constructFL output', file=sys.stderr)\n"
    "    else:\n"
    "        plog('Adjust FL-circRNA: adjFL')\n"
    "        adjFL([genome,anno,RG_outPrefix,thread])\n"
    "        plog('Adjust mistaken circRNA: circSeq')\n"
    "        circSeq(genome,RG_outPrefix,thread)\n"
    "        plog('Annotate circRNA: annotateNormal')\n"
    "        annotateNormal([RG_outPrefix,anno])\n"
)
if old_block in src:
    src = src.replace(old_block, new_block)
    print("  OK  : adjFL/circSeq/annotateNormal guard")
else:
    print("  WARN: adjFL block not found — already patched or changed?", file=sys.stderr)

# ---- c) circFL_output guard ----
old_out = (
    "    plog('Format circFL results: circFL_output')\n"
    "    circFL_output(RG_outPrefix)\n"
)
new_out = (
    "    if os.path.exists(RG_outPrefix+'result_Normal.txt'):\n"
    "        plog('Format circFL results: circFL_output')\n"
    "        circFL_output(RG_outPrefix)\n"
    "    else:\n"
    "        import sys; print('Skipping circFL_output: no result_Normal.txt', file=sys.stderr)\n"
)
if old_out in src:
    src = src.replace(old_out, new_out)
    print("  OK  : circFL_output guard")
else:
    print("  WARN: circFL_output line not found — already patched or changed?", file=sys.stderr)

with open(path, 'w') as f:
    f.write(src)
print("Patched RG.py OK")
EOF

# ── Patch 3: mRG_merge.py ─────────────────────────────────────────────────────
# set(iso2read.keys()) is non-deterministic; replace with list() for stable order.
RUN python3 - <<'EOF'
import sys

path = '/opt/conda/envs/circFL/lib/python3.9/site-packages/circfull/mRG_merge.py'
with open(path) as f:
    src = f.read()

old = 'set(iso2read.keys())'
new = 'list(iso2read.keys())'
if old in src:
    src = src.replace(old, new)
    print("Patched mRG_merge.py OK")
else:
    print("WARN: mRG_merge.py target not found — already patched or changed?", file=sys.stderr)

with open(path, 'w') as f:
    f.write(src)
EOF

# ── Patch 4: annoFL.py ────────────────────────────────────────────────────────
# Four related fixes for chromosomes/contigs absent from the GTF index:
#   a) After "if tmp['chr'] not in tabixfile.contigs:" return 'unannotated' immediately.
#   b) Wrap tabixfile.fetch() in try/except ValueError for out-of-range coordinates.
#   c) Add 'unannotated' to the recognised circ_type list.
#   d) Add the elif branch mapping circ_type 'unannotated' to n/m labels.
RUN python3 - <<'EOF'
import re, sys

path = '/opt/conda/envs/circFL/lib/python3.9/site-packages/circfull/annoFL.py'
with open(path) as f:
    lines = f.readlines()

patched_contig   = False
patched_fetch    = False
patched_typelist = False
patched_elif     = False

out = []
i = 0
while i < len(lines):
    line = lines[i]

    # ---- a) missing-contig early return ----
    if not patched_contig and "if tmp['chr'] not in tabixfile.contigs:" in line:
        indent = line[:len(line) - len(line.lstrip())]
        bi = indent + "    "
        out.append(line)
        out.append(bi + "start_type = ','.join(['unannotated'] * len(exon_start))\n")
        out.append(bi + "end_type   = ','.join(['unannotated'] * len(exon_end))\n")
        out.append(bi + "circ_type  = 'unannotated'\n")
        out.append(bi + "geneName   = ''\n")
        out.append(bi + "return (start_type, end_type, circ_type, geneName)\n")
        patched_contig = True
        i += 1
        continue

    # ---- b) ValueError guard around tabixfile.fetch ----
    if (not patched_fetch
            and "for gtf in tabixfile.fetch(" in line
            and "pysam.asGTF()" in line
            and "tmp['chr']" in line):
        indent = line[:len(line) - len(line.lstrip())]
        bi = indent + "    "
        m = re.search(r'for gtf in (tabixfile\.fetch\(.*\)):', line.strip())
        if m:
            fetch_expr = m.group(1)
            out.append(indent + "try:\n")
            out.append(bi     + "it = " + fetch_expr + "\n")
            out.append(indent + "except ValueError:\n")
            out.append(bi     + "start_type = ','.join(['unannotated'] * len(exon_start))\n")
            out.append(bi     + "end_type   = ','.join(['unannotated'] * len(exon_end))\n")
            out.append(bi     + "circ_type  = 'unannotated'\n")
            out.append(bi     + "geneName   = ''\n")
            out.append(bi     + "return (start_type, end_type, circ_type, geneName)\n")
            out.append(indent + "for gtf in it:\n")
            patched_fetch = True
        else:
            print("WARN: could not parse fetch() line, keeping original", file=sys.stderr)
            out.append(line)
        i += 1
        continue

    # ---- c) add 'unannotated' to recognised type list ----
    if not patched_typelist and "['intergenic','antisense','read through']" in line:
        line = line.replace(
            "['intergenic','antisense','read through']",
            "['intergenic','antisense','read through','unannotated']"
        )
        patched_typelist = True

    # ---- d) add elif branch for 'unannotated' after the 'intergenic' branch ----
    if not patched_elif and "elif i=='intergenic':" in line:
        indent = line[:len(line) - len(line.lstrip())]
        bi = indent + "    "
        out.append(line)           # elif i=='intergenic':
        i += 1
        for _ in range(2):         # copy the two body lines of the intergenic branch
            if i < len(lines):
                out.append(lines[i]); i += 1
        out.append(indent + "elif i == 'unannotated':\n")
        out.append(bi     + "n = 'unannotated'\n")
        out.append(bi     + "m = 'unannotated'\n")
        patched_elif = True
        continue

    out.append(line)
    i += 1

for flag, name in [
    (patched_contig,   "missing-contig early return"),
    (patched_fetch,    "ValueError guard around fetch()"),
    (patched_typelist, "'unannotated' added to type list"),
    (patched_elif,     "elif 'unannotated' branch"),
]:
    print(f"  {'OK  ' if flag else 'WARN'}: {name}", file=sys.stderr if not flag else sys.stdout)

with open(path, 'w') as f:
    f.writelines(out)
print("Patched annoFL.py OK")
EOF

ENV PATH="/opt/conda/envs/circFL/bin:/opt/conda/bin:$PATH"
ENTRYPOINT ["/opt/conda/envs/circFL/bin/circfull"]