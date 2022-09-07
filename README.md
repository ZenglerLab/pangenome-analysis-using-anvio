# pangenome-analysis-using-anvio
Pangenome analysis using the MerenLab's Anvio software

- Installation (under UNIX systems; sorry Windows users):

  Before to install Anvio, two things:
  - Anvio works perfectly with python3.9, but no with python3.10 (due to dependencies that are not working properly with python3.10/pyQT5)
  - Anvio's scripts are set to work with python3 aliased as `python`, not `python3`.
  
  So, to install Anvio, the better option is to install Anaconda3 that is still shipped with python3.9. Then install anvio from the Python Package index using `pip install anvio`. If you feel adventurous, you can download anvio from `https://github.com/merenlab/anvio` and make it work with python3.9 in a virtual environment.
  
- Now that you have installed python3.9/anvio, let's have fun:

Step00. (If and only if) Process contig files (e.g. `contig=<file basename>.fna`): Remove non ATCG and contigs shorter than 1000 nucleotides:

`anvi-script-reformat-fasta -o $(basename ${contig} .fna).anvio.fa $contig --seq-type NT -l 1000 --simplify-names`

Step01. Create an Anvio's CONTIG database for every contig file (parallelization at the bottom):

`mkdir -p input-anvio-contig-dbs && anvi-gen-contigs-database -f $contig -o input-anvio-contig-dbs/$(basename ${contig} .anvio.fa).db --project-name <project name> --force-overwrite --num-threads 1`

Step02. Gene annotation:

  - (Optional, although highly recommended) Annotate COGs for every gene call:

`anvi-run-ncbi-cogs -c $(basename ${contig} .anvio.fa).db --num-threads 1`

  - (Optional, although highly recommended) Annotate using Anvio's default HMMs:

`anvi-run-hmms -c $(basename ${contig} .anvio.fa).db --num-threads 1`

  - (Optional and requires the setup of the PFAM database) Annotate Protein Families:

`anvi-setup-pfams --pfam-data-dir <PFAM DIRECTORY>`

`anvi-run-pfams --contigs-db input-anvio-contig-dbs/$(basename ${contig} .anvio.fa).db --pfam-data-dir <PFAM DIRECTORY>`

  - (Optional and requires the setup of the SCG database) Taxonomy assignment:

`anvi-setup-scg-taxonomy --scgs-taxonomy-data-dir <SCG DIRECTORY>`

`anvi-run-scg-taxonomy --contigs-db input-anvio-contig-dbs/$(basename ${contig} .anvio.fa).db --scgs-taxonomy-data-dir <SCG DIRECTORY>`

  - (Optional and requires the setup of the tRNA database) Taxonomy assignment:

`anvi-setup-trna-taxonomy --trna-taxonomy-data-dir <tRNA DIRECTORY>`

`anvi-run-trna-taxonomy --contigs-db input-anvio-contig-dbs/$(basename ${contig} .anvio.fa).db --trna-taxonomy-data-dir <tRNA DIRECTORY>`

  - (Optional and requires the setup of the INTERACDOME database) InteracDome profiles:

`anvi-setup-interacdome --interacdome-data-dir <INTERACDOME DIRECTORY>`

`anvi-run-interacdome --contigs-db input-anvio-contig-dbs/$(basename ${contig} .anvio.fa).db -O INTERACDOME/$(basename ${contig} .fa) --interacdome-data-dir <INTERACDOME DIRECTORY>`

  - (Optional and requires the setup of the KEGG database) Call KEGG Orthology:

`anvi-setup-kegg-kofams --kegg-data-dir <KEGG DIRECTORY>`

`anvi-run-kegg-kofams --contigs-db input-anvio-contig-dbs/$(basename ${contig} .anvio.fa).db --kegg-data-dir <KEGG DIRECTORY>`

Step03. Create an Anvio's GENOME database. This step requires the manually setup of a `external-genomes.txt` file and the GENOME database filename must end with `-GENOME.db`. The `external-genomes.txt` file is a two-columns, tab-separated file with the header `name	contigs_db_path`:

`mkdir -p input-anvio-GENOMES.db && anvi-gen-genomes-storage -e external-genomes.txt -o input-anvio-GENOMES.db/<basename>-GENOMES.db`

Step04. Run the pangenome analysis. The `mcl-inflation` argument depends on how closely related the genomes are (see https://merenlab.org/2016/11/08/pangenomics-v2/), and the `--I-know-this-is-not-a-good-idea` is to force Anvio to analyze hundreds of genomes:

`anvi-pan-genome -g input-anvio-GENOMES.db/<basename>-GENOMES.db -n <pangenome name> --mcl-inflation 10 --force-overwrite --I-know-this-is-not-a-good-idea`

Step05a. Extract single-copy core genes (individual genes and concatenated gene alignments). Change the number of genomes per gene cluster (`X`):

`anvi-get-sequences-for-gene-clusters -p <pangenome name>/<pangenome name>-PAN.db -g input-anvio-GENOMES.db/<basename>-GENOMES.db --min-num-genomes-gene-cluster-occurs X --max-num-genes-from-each-genome 1 --output-file <pangenome name>/<pangenome name>-SCGs.fa`

`anvi-get-sequences-for-gene-clusters -p <pangenome name>/<pangenome name>-PAN.db -g input-anvio-GENOMES.db/<basename>-GENOMES.db --min-num-genomes-gene-cluster-occurs X --max-num-genes-from-each-genome 1 --concatenate-gene-clusters --output-file <pangenome name>/<pangenome name>-aligned-SCGs.fa`

Step05b. Add collection of gene clusters to the pangenome:

`anvi-get-sequences-for-gene-clusters -p <pangenome name>/<pangenome name>-PAN.db -g input-anvio-GENOMES.db/<basename>-GENOMES.db -C <collection name>`

Step06. Interactive visualization:

`anvi-display-pan -p <pangenome name>/<pangenome name>-PAN.db -g input-anvio-GENOMES.db/<basename>-GENOMES.db`

Step07. Summarize and extract data for further analyses (pangenome curves, gene cluster heatmaps, etc). More information here https://anvio.org/help/main/programs/anvi-summarize/:

`anvi-script-add-default-collection -p <pangenome name>/<pangenome name>-PAN.db -C DEFAULT`
`anvi-summarize -p <pangenome name>/<pangenome name>-PAN.db -g input-anvio-GENOMES.db/<basename>-GENOMES.db -C DEFAULT -o <pangenome name>/SUMMARY`

Step08. Determine similarity between genomes

`anvi-compute-genome-similarity -e external-genomes.txt -p <pangenome name>/<pangenome name>-PAN.db -o <ANI directory output> --program fastANI`

Step09. Phylogeny based on SCGs (using `X` threads and it requires a manually setup of a `layer-order.txt` file):

`trimal -in <pangenome name>/<pangenome name>-aligned-SCGs.fa -out <pangenome name>/<pangenome name>-SCGs-clean.fa -gt 0.50`

`iqtree2 -s <pangenome name>/<pangenome name>-aligned-SCGs-clean.fa -T X --mset WAG --ufboot 1000`

`echo -e "item_name\tdata_type\tdata_value" > <pangenome name>/layer-order.txt`

`` echo -e "SCGs_Bayesian_Tree\tnewick\t`cat <pangenome name>/<pangenome name>-aligned-SCGs-clean.fa.contree`" >> <pangenome name>/layer-order.txt ``

`anvi-import-misc-data -p <pangenome name>/<pangenome name>-PAN.db -t layer_orders <pangenome name>/layer-order.txt`

Step10. Estimate metabolic coverage of CONTIG databases:

`anvi-estimate-metabolism --contigs-db input-anvio-contig-dbs/$(basename ${contig} .anvio.fa).db -O KEGG/$(basename ${contig} .fa)`

Final remarks:

- If you installed a virtual environment, you need to setup the variables `$PATH` and `$PYTHONPATH`:

```
source /opt/python3-venv/p39/bin/activate
export PATH=$PATH:/opt/git-repositories/anvio.merenlab/bin
export PATH=$PATH:/opt/git-repositories/anvio.merenlab/sandbox
export PYTHONPATH=$PYTHONPATH:/opt/git-repositories/anvio.merenlab
export PATH=$PATH:/opt/repositories/git-reps/FastANI.ParBLiSS
export PATH=$PATH:/opt/repositories/git-reps/trimal.scapella/source
export PATH=/opt/repositories/git-reps/diamond.bbuchfink/bin:$PATH
```

- To parallelize using `bash` (not `sh`):

```
for contig in $(find <directory with nucleotide fasta files> -depth -name "*fna" -name "*fa")
  do ((j=j%8)); ((j++==0)) && wait;
  <command here> &
  done
```
