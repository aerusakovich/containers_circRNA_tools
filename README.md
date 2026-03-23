# circRNA Long-Read Sequencing Containers

A collection of containerized pipelines for circRNA detection and characterization from long-read sequencing data. All containers are available at quya.io.

Each container ships with pre-installed dependencies and, where applicable, applies upstream bug fixes that commonly break tools during real runs.

---

## Containers

###  CIRI-long

Full-length circRNA detection from long-read sequencing data.

```
docker pull quay.io/anrusakovich/ciri-long
```

- **Python 3.8** with git, make, build-essential
- CIRI-long pre-installed via pip

```bash
singularity exec --bind /your/data:/data ciri-long.sif \
    ciri-long <subcommand> [options]
```

> ⚠️ **AVX2 required.** Run on AVX2-compatible hardware (e.g. `--constraint=avx2` in Slurm).
> ⚠️ **Core count.** Providing too many cores may cause OOM errors — try reducing available cores if this occurs despite sufficient memory.

Original tool: https://github.com/bioinfo-biols/CIRI-long

---

### circFL-seq (circfull)

Full-length circRNA detection and annotation.

```
docker pull quay.io/anrusakovich/circfl-seq
```

- **Python 3.9** conda environment with minimap2, samtools, bedtools, TRF, TideHunter + Python dependencies
- circfull pre-installed

This image applies upstream bug/compatibility fixes that commonly break the tool during real runs:

| Fix | Description |
|-----|-------------|
| RG | Fixes incorrect `filterOut()` argument wiring (prevents wrong fastq/genome arguments) |
| mRG merge | Pandas compatibility - set indexer changed to list |
| DNSC getNovo | Handles empty TideHunter output (no silent crash; emits warning and empty Pass file) |
| Annotation | Prevents Tabix fetch crashes when genome contains contigs absent from the GTF; labels such events as `unannotated` instead of crashing |

```bash
singularity exec --bind /your/data:/data circfull.sif \
    circfull <subcommand> [options]
```

> ⚠️ **AVX2 required.** Run on AVX2-compatible hardware (e.g. `--constraint=avx2` in Slurm).

Original tool: https://github.com/yangence/circfull

---

### isoCirc

Full-length circRNA isoform characterization from long-read sequencing data.

```
docker pull quay.io/anrusakovich/isocirc
```

- **Python 3.11** with git, gcc, make, build-essential, gawk
- isoCirc pre-installed via pip
- bedtools ≥ 2.27.0 and minimap2 ≥ 2.11 installed via conda

```bash
singularity exec --bind /your/data:/data isocirc.sif \
    isocirc <fastq> <genome.fa> <annotation.gtf> <circRNA.bed> <output_dir> [options]
```

Original tool: https://github.com/Xinglab/isoCirc

---

### CircNick-LRS

circRNA detection in nanopore long-read sequencing data.

```
docker pull quay.io/anrusakovich/circnick-lrs
```

- **Python 3.7.6** micromamba environment with bedtools=2.29.2, samtools=1.9, pblat=2.5, NanoFilt, deeptools + all Python dependencies
- `long_read_circRNA v2.1` pre-installed
- Reference data for **human and mouse (mm10/GRCm38)** pre-downloaded and bundled at `/opt/long_read_circRNA/data`
- All genome indexes pre-built at container build time (samtools `.fai`, pblat `.flat`/`.gdx`) — no indexing or reference setup required at runtime

This image applies upstream bug/compatibility fixes that commonly affect the tool during real runs:

| Fix | Description |
|-----|-------------|
| NumPy compatibility | Pins numpy<2 to prevent deeptools/bamCoverage crash (`_ARRAY_API not found`) |
| BAM naming mismatch | Fixes `scan.circRNA.sort.bam` → `circRNA.sort.bam` in `extra_stuff_v2.sh` and `novel_exons_and_alternative_usage_v7.0.sh` (prevents intron coverage step from silently failing) |
| Empty novel exon guard | Adds file size checks before `mapBed` and `bedtools getfasta` calls on `novel.exons.2reads.bed` to prevent bedtools column errors when no novel exons with 2+ read support are detected |

```bash
singularity exec \
    --bind /your/data:/data \
    circnick-lrs.sif \
    long_read_circRNA run \
    --species mouse \
    --reference-path /opt/long_read_circRNA/data \
    --script-path /opt/long_read_circRNA/scripts \
    --output-path /data/output \
    /data/sample.fq.gz
```

Original tool: https://github.com/omiics-dk/long_read_circRNA

---

## General Usage Notes

- **Bind-mount** your input FASTQ, genome, annotation, and output directories into the container as needed using `--bind /host/path:/container/path` (Singularity) or `-v /host/path:/container/path` (Docker).
- All containers are tested on Linux x86_64.
- For HPC/Slurm environments, Singularity is recommended over Docker.
