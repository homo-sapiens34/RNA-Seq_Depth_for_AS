# RNA-Seq_Depth_for_AS

How deep does an RNA-Seq library have to be before it stops finding new
alternative splicing?

Around 200M mapped reads is where the cost of each additional detection starts
to climb faster than the return. Lowly expressed genes are still gaining at
150M; genes above 10 TPM level off between 100M and 150M. See
`notebooks/02_downsampled_cohorts.ipynb` and `05_sequencing_cost.ipynb`.

**Sequencing depth** always means mapped reads, the 50-300M downsampling axis,
never per-junction read support. **Genes with AS** and **junctions involved in
AS** are MAJIQ local splicing variants with 0.05 < PSI < 0.95. Each sample was
downsampled 100 times per depth; a junction found in all 100 subsamples is
**robust**, in fewer **sporadic**. An **expression category** is a TPM bin.

## Layout

```
notebooks/    the analysis, one notebook per question
asdepth/      readers for the MAJIQ summary tables, plot styling
scripts/      the command-line steps that build data/derived/
data/         inputs; nothing here is written by this repository
data/derived/ tables built by scripts/, small enough to keep in git
results/      figures and tables written by the notebooks
tests/        invariants the analysis has to satisfy
webapp/       source of sequencing-depth4splicing.com
```

Flow is one way: scripts write `data/derived/`, notebooks read `data/` and write
`results/`. Nothing under `results/` is read back in. Only file reading and plot
styling live in `asdepth`; the statistics stay in the notebooks.

Each notebook locates the top of the clone itself, so it runs from anywhere
inside it:

```
python3 -m nbconvert --to notebook --execute --inplace notebooks/04_gtex_and_tcga.ipynb
```

## The notebooks

| Notebook | Question | Download | Runs in |
|---|---|---|---|
| `01_sars_cov_2_cohort` | Detections per depth in a cohort sequenced 60-150M | yes | ~5 min |
| `02_downsampled_cohorts` | The same, downsampled 100x per depth to 300M | yes | ~15 min |
| `03_junction_features` | Which kinds of junction do the extra reads add? | no | seconds |
| `04_gtex_and_tcga` | How much do GTEx and TCGA miss at their usual depth? | no | seconds |
| `05_sequencing_cost` | What does each additional detection cost? | no | seconds |
| `06_detectability_model` | Which gene properties predict detection? | no | seconds |
| `07_biological_heterogeneity` | Does heterogeneity rather than depth drive discovery? | yes | ~1 min |
| `08_pathway_enrichment` | Are the pathways found only at 200M coherent? | no | seconds |

## Data

Four notebooks run straight after a clone. The other four need the
`*_summary.tsv` splicing tables, which are on Zenodo,
[10.5281/zenodo.11655945](https://doi.org/10.5281/zenodo.11655945), together
with the raw MAJIQ output. `07_biological_heterogeneity` also needs the
rnaseqmut output. Each notebook names the paths it expects in its first cell.

The junction feature table behind `03_junction_features` and
`06_detectability_model` is not distributed: it is a gigabyte, and
`scripts/build_junction_feature_table.py` builds it from those summary tables,
a GENCODE annotation and `data/annotation/`. Its two summaries are in
`data/derived/`, so neither notebook needs it.

The DICAST-unified event tables that NEASE was run on are not distributed
either. The enrichment output is in `data/derived/nease/`, which is what
`08_pathway_enrichment` reads.

`data/README.md` says where every shipped file comes from. One deserves
attention: `expression_category_totals.tsv` is the denominator that turns
detection counts into percentages. It counts *all* expressed genes in a TPM bin,
not only those with alternative splicing, so it comes from the expression
matrices rather than from the splicing analysis, and a percentage against it
mixes depth-limited detection with genes that only ever have one isoform.

## Building the inputs

```
bash    scripts/har_genes.sh                     # genes overlapping HARs
python3 scripts/build_junction_feature_table.py  # the junction feature table
python3 scripts/summarise_junction_features.py   # its two small summaries
python3 scripts/fit_gee_models.py                # detectability coefficients
```

`tpm.py` normalises featureCounts output. `count_genes_from_majiq.py` and
`count_junctions_from_majiq.py` produce the count files under `data/derived/`.
`make_depth_estimator_coeffs.py` builds `webapp/data/seq_depth_coeffs.json`.

## Checks

```
python3 -m pytest tests
```

Invariants rather than stored numbers: proportions sum to one, detections and
cost per detection rise with depth, junctions are cheaper to find than genes,
the detectability model agrees with the composition analysis. One check reads a
miniature summary file counted by hand. CI runs them on every push with the
notebooks that need no download.

## Requirements

Python 3.10 to 3.12 and `requirements.txt`; the pinned numpy does not build on
3.13.

MIT licensed, see [LICENSE](LICENSE). Citation metadata in `CITATION.cff`.
