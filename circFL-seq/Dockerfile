FROM condaforge/mambaforge:latest
LABEL authors="Anastasia Rusakovich"
LABEL description="Docker image for circFL with custom patches and specific dependency versions"

ENV DEBIAN_FRONTEND=noninteractive
ENV LC_ALL=C
ENV PYTHON_UNBUFFERED=1

ENV PATH="/opt/conda/envs/circFL/bin:$PATH"

RUN mamba create -y -n circFL -c conda-forge -c bioconda \
    python=3.9 samtools bcftools minimap2 bedtools trf \
    tidehunter pysam numpy pandas=1.5.3 portion \
    scikit-learn pyfasta mappy progressbar docopt porechop && \
    /opt/conda/envs/circFL/bin/pip install circfull && \
    chmod +x /opt/conda/envs/circFL/bin/* && \

    PKG_DIR="/opt/conda/envs/circFL/lib/python3.9/site-packages/circfull" && \
    FILE="${PKG_DIR}/annoFL.py" && \
    sed -i "s/filterOut(RG_outPrefix,RG_outPrefix,thread,rmskFile)/filterOut(RG_outPrefix,RG_outPrefix,fastq,genome,thread,rmskFile)/" ${PKG_DIR}/RG.py && \
    sed -i "s/set(iso2read.keys())/list(iso2read.keys())/g" ${PKG_DIR}/mRG_merge.py && \
    sed -i "/thFile=pd.read_csv/a \    if thFile.empty: print('#### Warning: DNSC - TideHunter.tab is empty ####', file=sys.stderr); thFile.to_csv(outPrefix+'TideHunter_Pass.tab',sep='\\\\t',index=False); return" ${PKG_DIR}/DNSC_getNovo.py && \
    sed -i "/if tmp\['chr'\] not in tabixfile\.contigs:/a \\
        start_type=','.join(['unannotated']* len(exon_start))\\
        end_type=','.join(['unannotated']* len(exon_end))\\
        circ_type='unannotated'\\
        geneName=''\\
        return(start_type,end_type,circ_type,geneName)" "$FILE" && \
    sed -i "/for gtf in tabixfile\.fetch(tmp\['chr'\], tmp\['start'\]-1, tmp\['end'\]-1,parser=pysam\.asGTF())\:/c\\
    try:\\
        it = tabixfile.fetch(tmp['chr'], tmp['start']-1, tmp['end']-1,parser=pysam.asGTF())\\
    except ValueError:\\
        start_type=','.join(['unannotated']* len(exon_start))\\
        end_type=','.join(['unannotated']* len(exon_end))\\
        circ_type='unannotated'\\
        geneName=''\\
        return(start_type,end_type,circ_type,geneName)\\
    for gtf in it:" "$FILE" && \
    sed -i "s/\['intergenic','antisense','read through'\]/['intergenic','antisense','read through','unannotated']/g" "$FILE" && \
    sed -i "/elif i=='intergenic':/{n; n; a\\
        elif i=='unannotated':\\
            n='unannotated'\\
            m='unannotated'
    }" "$FILE" && \

    conda clean -afy

ENTRYPOINT ["/opt/conda/envs/circFL/bin/circfull"]