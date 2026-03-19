# syntax=docker/dockerfile:1
FROM condaforge/mambaforge:latest
LABEL authors="anrusakovich"
LABEL description="Docker image for circFL-seq (patched)"

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
for line in lines:
    if not patched and "FL_result['len']=FL_result.apply(getExonLen" in line.replace(" ", ""):
        indent = line[:len(line) - len(line.lstrip())]
        new_lines.append(indent + "if FL_result.empty:\n")
        new_lines.append(indent + "    import sys; print('cRG constructFL: no valid FL-circRNAs, skipping', file=sys.stderr)\n")
        new_lines.append(indent + "    return\n")
        patched = True
    new_lines.append(line)

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

if 'set(iso2read.keys())' in src:
    src = src.replace('set(iso2read.keys())', 'list(iso2read.keys())')
    print("Patched mRG_merge.py OK")
else:
    print("WARN: mRG_merge.py target not found — already patched or changed?", file=sys.stderr)

with open(path, 'w') as f:
    f.write(src)
EOF

# ── Patch 4: annoFL.py ────────────────────────────────────────────────────────
# Three fixes for chromosomes absent from or out-of-range in the GTF index.
# NOTE: the pip-installed version has NO existing "tabixfile.contigs" check,
# so (a) is inserted fresh at the fetch() call-site rather than appended to
# an existing guard.
#
#   a) Insert "if chr not in tabixfile.contigs: return unannotated" immediately
#      BEFORE the tabixfile.fetch() for-loop — trigger: "tabixfile.fetch(" AND
#      "pysam.asGTF()" on the same line.
#   b) Wrap that same fetch() in try/except ValueError for out-of-range coords.
#   c) Add 'unannotated' to the recognised circ_type list.
#   d) Add the elif branch mapping circ_type 'unannotated' to n/m labels.
RUN python3 - <<'EOF'
import re, sys

path = '/opt/conda/envs/circFL/lib/python3.9/site-packages/circfull/annoFL.py'
with open(path) as f:
    lines = f.readlines()

# Diagnostic: show every line that mentions 'fetch' or 'contig' so mismatches
# are immediately visible in the build log.
print("=== annoFL.py fetch/contig lines ===", file=sys.stderr)
for n, l in enumerate(lines):
    if "fetch" in l or "contig" in l.lower():
        print(f"  {n:4d}: {l!r}", file=sys.stderr)
print("=====================================", file=sys.stderr)

patched_fetch    = False   # covers both (a) contig-guard and (b) ValueError wrap
patched_typelist = False
patched_elif     = False

out = []
i = 0
while i < len(lines):
    line = lines[i]

    # ---- a+b) insert contig guard then wrap fetch() in ValueError handler ----
    # Both are injected at the same site: the "for X in tabixfile.fetch(...)" line.
    if (not patched_fetch
            and "tabixfile.fetch(" in line
            and "pysam.asGTF()" in line):
        indent = line[:len(line) - len(line.lstrip())]
        bi = indent + "    "
        m = re.search(r'for \w+ in (tabixfile\.fetch\(.*\)):', line)
        if m:
            fetch_expr = m.group(1)
            # (a) contig guard — inserted fresh since pip version lacks it
            out.append(indent + "if tmp['chr'] not in tabixfile.contigs:\n")
            out.append(bi     + "start_type = ','.join(['unannotated'] * len(exon_start))\n")
            out.append(bi     + "end_type   = ','.join(['unannotated'] * len(exon_end))\n")
            out.append(bi     + "circ_type  = 'unannotated'\n")
            out.append(bi     + "geneName   = ''\n")
            out.append(bi     + "return (start_type, end_type, circ_type, geneName)\n")
            # (b) ValueError guard wrapping the original fetch
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
            print(f"WARN: fetch() regex did not match — raw line: {line!r}", file=sys.stderr)
            out.append(line)
        i += 1
        continue

    # ---- c) add 'unannotated' to recognised type list ----
    if not patched_typelist and (
            "['intergenic','antisense','read through']" in line or
            '["intergenic","antisense","read through"]' in line):
        line = line.replace(
            "['intergenic','antisense','read through']",
            "['intergenic','antisense','read through','unannotated']"
        ).replace(
            '["intergenic","antisense","read through"]',
            '["intergenic","antisense","read through","unannotated"]'
        )
        patched_typelist = True

    # ---- d) elif 'unannotated' branch after the 'intergenic' branch ----
    if not patched_elif and (
            "elif i=='intergenic':" in line or
            'elif i=="intergenic":' in line):
        indent = line[:len(line) - len(line.lstrip())]
        bi = indent + "    "
        out.append(line)      # keep the elif i=='intergenic': line
        i += 1
        for _ in range(2):    # copy the two body lines of the intergenic branch
            if i < len(lines):
                out.append(lines[i]); i += 1
        out.append(indent + "elif i == 'unannotated':\n")
        out.append(bi     + "n = 'unannotated'\n")
        out.append(bi     + "m = 'unannotated'\n")
        patched_elif = True
        continue

    out.append(line)
    i += 1

# Final report
all_ok = True
for flag, name in [
    (patched_fetch,    "contig guard + ValueError wrap at fetch() site"),
    (patched_typelist, "'unannotated' added to type list"),
    (patched_elif,     "elif 'unannotated' branch"),
]:
    if flag:
        print(f"  OK  : {name}")
    else:
        print(f"  WARN: {name} — target not found; see diagnostic dump above", file=sys.stderr)
        all_ok = False

with open(path, 'w') as f:
    f.writelines(out)
print("Patched annoFL.py OK")
EOF

ENV PATH="/opt/conda/envs/circFL/bin:/opt/conda/bin:$PATH"
ENTRYPOINT ["/opt/conda/envs/circFL/bin/circfull"]