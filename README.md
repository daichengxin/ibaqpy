# ibaqpy

[![Python application](https://github.com/bigbio/ibaqpy/actions/workflows/python-app.yml/badge.svg)](https://github.com/bigbio/ibaqpy/actions/workflows/python-app.yml)
[![Upload Python Package](https://github.com/bigbio/ibaqpy/actions/workflows/python-publish.yml/badge.svg)](https://github.com/bigbio/ibaqpy/actions/workflows/python-publish.yml)
[![Codacy Badge](https://app.codacy.com/project/badge/Grade/6a1961c7d57c4225b4891f73d58cac6b)](https://app.codacy.com/gh/bigbio/ibaqpy/dashboard?utm_source=gh&utm_medium=referral&utm_content=&utm_campaign=Badge_grade)
[![PyPI version](https://badge.fury.io/py/ibaqpy.svg)](https://badge.fury.io/py/ibaqpy)
![PyPI - Downloads](https://img.shields.io/pypi/dm/ibaqpy)

iBAQ (intensity Based Absolute Quantification) determines the abundance of a protein by dividing the total precursor intensities by the number of theoretically observable peptides of the protein. The TPA (Total Protein Approach) value is determined by summing peptide intensities of each protein and then dividing by the molecular mass to determine the relative concentration of each protein. By using [ProteomicRuler](https://www.sciencedirect.com/science/article/pii/S1535947620337749), it is possible to calculate the protein copy number and absolute concentration. **ibaqpy** compute IBAQ values, TPA values, copy numbers and concentration for proteins starting from a msstats input file (or a feature parquet from [quantmsio](https://github.com/bigbio/quantms.io)) and a SDRF experimental design file. In addition, it supports the merging of iBAQ results from multiple datasets and the elimination of outliers and batch effects. This package provides multiple tools: 

- `peptide_normalization.py`: Generate the peptides dataframe from a msstats input file and a SDRF experimental design file (or directly from a feature parquet), then normalize the peptides dataframe. It includes multiple steps such as peptidoform normalization, peptidorom to peptide summarization, peptide intensity normalization, and imputation. 

- `compute_ibaq.py`: Compute IBAQ values from the output file from script `peptide_normalization.py`.

- `compute_tpa.py`: Compute TPA values, protein copy numbers and concentration from the output file from script `peptide_normalization.py`.

- `datasets_merge.py`: Merge ibaq results from multiple datasets. It consists of three steps: missing value imputation, outlier removal, and batch effect removal.

**NOTE:** In all scripts and result files, *uniprot accession* is used as the protein identifier.

### How to install ibaqpy

Ibaqpy is available in PyPI and can be installed using pip:

```asciidoc
pip install ibaqpy
```

You can install the package from code: 

1. Clone the repository:

```asciidoc
>$ git clone https://github.com/bigbio/ibaqpy
>$ cd ibaqpy
```

2. Install conda environment:

```asciidoc
>$ mamba env create -f conda-environment.yaml
```

3. Install ibaqpy:

```asciidoc
>$ python setup.py install
```

### Collecting intensity files 

Absolute quantification files has been store in the following url: 

```
https://ftp.pride.ebi.ac.uk/pub/databases/pride/resources/proteomes/absolute-expression/
```

Inside each project reanalysis folder, the folder proteomicslfq contains the msstats input file with the structure `{Name of the project}_msstats_in.csv`. 

E.g. http://ftp.pride.ebi.ac.uk/pub/databases/pride/resources/proteomes/absolute-expression/PXD003947/proteomicslfq/PXD003947.sdrf_openms_design_msstats_in.csv 

### Peptide Normalization - peptide_normalization.py

```asciidoc
python peptide_normalization.py --msstats PXD003947.sdrf_openms_design_msstats_in.csv --sdrf PXD003947.sdrf.tsv --remove_ids data/contaminants_ids.tsv --remove_decoy_contaminants --remove_low_frequency_peptides --output PXD003947-peptides-norm.csv
``` 

The command provides an additional `flag` for skip_normalization, pnormalization, compress, log2, violin, verbose. If you use feature parquet as input, you can pass the `--sdrf`.

```asciidoc
Usage: peptide_normalization.py [OPTIONS]

Options:
  -m, --msstats TEXT              MsStats file import generated by quantms
  -p, --parquet TEXT              Parquet file import generated by quantmsio
  -s, --sdrf TEXT                 SDRF file import generated by quantms
  --stream                        Stream processing normalization
  --chunksize                     The number of rows of MSstats or parquet read using pandas streaming
  --min_aa INTEGER                Minimum number of amino acids to filter
                                  peptides
  --min_unique INTEGER            Minimum number of unique peptides to filter
                                  proteins
  --remove_ids TEXT               Remove specific protein ids from the
                                  analysis using a file with one id per line
  --remove_decoy_contaminants     Remove decoy and contaminants proteins from
                                  the analysis
  --remove_low_frequency_peptides Remove peptides that are present in less
                                  than 20% of the samples
  --output TEXT                   Peptide intensity file including other all
                                  properties for normalization
  --skip_normalization            Skip normalization step
  --nmethod TEXT                  Normalization method used to normalize
                                  intensities for all samples (options: msstats, quantile, qnorm)
  --pnormalization                Normalize the peptide intensities using
                                  different methods (options: quantile, qnorm)
  --compress                      Read the input peptides file in compress
                                  gzip file
  --log2                          Transform to log2 the peptide intensity
                                  values before normalization
  --violin                        Use violin plot instead of boxplot for
                                  distribution representations
  --verbose                       Print addition information about the
                                  distributions of the intensities, number of
                                  peptides remove after normalization, etc.
  --qc_report TEXT                PDF file to store multiple QC images
  --help                          Show this message and exit.
```

Peptide normalization starts from the peptides dataframe. The structure of the input contains the following columns: 

- ProteinName: Protein name
- PeptideSequence: Peptide sequence including post-translation modifications `(e.g. .(Acetyl)ASPDWGYDDKN(Deamidated)GPEQWSK)`
- PrecursorCharge: Precursor charge
- FragmentIon: Fragment ion
- ProductCharge: Product charge
- IsotopeLabelType: Isotope label type
- Condition: Condition label `(e.g. heart)`
- BioReplicate: Biological replicate index `(e.g. 1)`
- Run: Run index `(e.g. 1)`
- Fraction: Fraction index `(e.g. 1)`
- Intensity: Peptide intensity
- Reference: Name of the RAW file containing the peptide intensity `(e.g. Adult_Heart_Gel_Elite_54_f16)`
- SampleID: Sample ID `(e.g. PXD003947-Sample-3)`
- StudyID: Study ID `(e.g. PXD003947)`. In most of the cases the study ID is the same as the ProteomeXchange ID.

#### 1. Removing Contaminants and Decoys

The first step is to remove contaminants and decoys. The script `peptide_normalization.py` provides a parameter `--remove_decoy_contaminants` as a flag to remove all the proteins with the following prefixes: `CONTAMINANT` and `DECOY`. And the user can provide a file with a list of protein accessions which represent each contaminant or high abundant protein in the file. An example file can be seen in `data/contaminants_ids.txt`.

#### 2. Peptidoform Normalization

A peptidoform is a combination of a `PeptideSequence(Modifications) + Charge + BioReplicate + Fraction`. In the current version of the file, each row correspond to one peptidoform. 

The current version of the tool uses the parackage [qnorm](https://pypi.org/project/qnorm/) to normalize the intensities for each peptidofrom. **qnorm** implements a quantile normalization method. However, the current version of qnorm can not handle NA values which will lead to cause more NA values in data. We suggest users to use default method 'quantile' instead for now.

#### 3. Peptidoform to Peptide Summarization

For each peptidoform a peptide sequence (canonical) with not modification is generated. The intensity of all peptides group by biological replicate are `sum`. 

Then, the intensities of the peptides across different biological replicates are summarize using the function `median`. 

At the end of this step, for each peptide, the corresponding protein + the intensity of the peptide is stored. 

#### 4. Peptide Intensity Imputation and Normalization

Before the final two steps (peptide normalization and imputation), the algorithm removes all peptides that are source of missing values significantly. The algorithm removes all peptides that have more than 80% of missing values and peptides that do not appear in more than 1 sample. 

Finally, two extra steps are performed: 

- ``peptide intensity imputation``: Imputation is performed using the package [missingpy](https://pypi.org/project/missingpy/). The algorithm uses a Random Forest algorithm to perform the imputation.
- ``peptide intensity normalization``: Similar to the normalization of the peptidoform intensities, the peptide intensities are normalized using the package [qnorm](https://pypi.org/project/qnorm/).

### Compute IBAQ - compute_ibaq.py
IBAQ is an absolute quantitative method based on strength that can be used to estimate the relative abundance of proteins in a sample. IBAQ value is the total intensity of a protein divided by the number of theoretical peptides.

```asciidoc
python compute_ibaq.py --fasta Homo-sapiens-uniprot-reviewed-contaminants-decoy-202210.fasta --peptides PXD003947-peptides.csv --enzyme "Trypsin" --normalize --output PXD003947-ibaq.tsv
``` 

The command provides an additional `flag` for normalize IBAQ values.

```asciidoc
python compute_ibaq.py --help
Usage: compute_ibaq.py [OPTIONS]

  Compute the IBAQ values for a file output of peptides with the format described in
  peptide_normalization.py.

  :param min_aa: Minimum number of amino acids to consider a peptide
  :param max_aa: Maximum number of amino acids to consider a peptide
  :param fasta: Fasta file used to perform the peptide identification
  :param peptides: Peptide intensity file
  :param enzyme: Enzyme used to digest the protein sample
  :param normalize: use some basic normalization steps
  :param output: output format containing the ibaq values
  :param verbose: Print addition information about the distributions of the intensities, 
                  number of peptides remove after normalization, etc.
  :param qc_report: PDF file to store multiple QC images

Options:
  -f, --fasta TEXT      Protein database to compute IBAQ values  [required]
  -p, --peptides TEXT   Peptide identifications with intensities following the peptide intensity output  [required]
  -e, --enzyme          Enzyme used during the analysis of the dataset (default: Trypsin)
  -n, --normalize       Normalize IBAQ values using by using the total IBAQ of the experiment
  --min_aa              Minimum number of amino acids to consider a peptide (default: 7)
  --max_aa              Maximum number of amino acids to consider a peptide (default: 30)
  -o, --output TEXT     Output format containing the ibaq values
  --verbose             Print addition information about the distributions of the intensities, 
                        number of peptides remove after normalization, etc.
  --qc_report           PDF file to store multiple QC images (default: "IBAQ-QCprofile.pdf")
  --help                Show this message and exit.
```

#### 1. Performs the Enzymatic Digestion
The current version of this tool uses OpenMS method to load fasta file, and use [ProteaseDigestion](https://openms.de/current_doxygen/html/classOpenMS_1_1ProteaseDigestion.html) to enzyme digestion of protein sequences, and finally get the theoretical peptide number of each protein.

#### 2. Calculate the IBAQ Value
First, peptide intensity dataframe was grouped according to protein name, sample name and condition. The protein intensity of each group was summed. Finally, the sum of the intensity of the protein is divided by the number of theoretical peptides.

If protein-group exists in the peptide intensity dataframe, the intensity of all proteins in the protein-group is summed based on the above steps, and then multiplied by the number of proteins in the protein-group.

#### 3. IBAQ Normalization  
Normalize the ibaq values using the total ibaq of the sample. The resulted ibaq values are then multiplied by 100'000'000 (PRIDE database noramalization), for the ibaq ppb and log10 shifted by 10 (ProteomicsDB).

### Compute TPA - compute_tpa.py
The total protein approach (TPA) is a label- and standard-free method for absolute protein quantitation of proteins using large-scale proteomic data. In the current version of the tool, the TPA value is the total intensity of the protein divided by its theoretical molecular mass.

[ProteomicRuler](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC4256500/) is a method for protein copy number and concentration estimation that does not require the use of isotope labeled standards. It uses the mass spectrum signal of histones as a "proteomic ruler" because it is proportional to the amount of DNA in the sample, which in turn depends on cell count. Thus, this approach can add an absolute scale to the mass spectrometry readout and allow estimates of the copy number of individual proteins in each cell.

```asciidoc
python compute_tpa.py --fasta Homo-sapiens-uniprot-reviewed-contaminants-decoy-202210.fasta --organism 'human' --peptides PXD003947-peptides.csv --ruler --ploidy 2 --cpc 200 --output PXD003947-tpa.tsv --verbose
```

```asciidoc
python compute_tpa.py --help
Usage: compute_tpa.py [OPTIONS]

  Compute the protein copy numbers and concentrations according to a file output of peptides with the
  format described in peptide_normalization.py.

  :param fasta: Fasta file used to perform the peptide identification
  :param peptides: Peptide intensity file
  :param organism: Organism source of the data
  :param ruler: Whether to compute protein copy number, weight and concentration.
  :param ploidy: Ploidy number
  :param cpc: Cellular protein concentration(g/L)
  :param output: Output format containing the TPA values, protein copy numbers and concentrations
  :param verbose: Print addition information about the distributions of the intensities, 
                  number of peptides remove after normalization, etc.
  :param qc_report: PDF file to store multiple QC images

Options:
  -f, --fasta TEXT      Protein database to compute IBAQ values  [required]
  -p, --peptides TEXT   Peptide identifications with intensities following the peptide intensity output  [required]
  -m, --organism        Organism source of the data.
  -r, --ruler           Calculate protein copy number and concentration according to ProteomicRuler
  -n, --ploidy          Ploidy number (default: 2)
  -c, --cpc             Cellular protein concentration(g/L) (default: 200)
  -o, --output TEXT     Output format containing the TPA values, protein copy numbers and concentrations
  --verbose             Print addition information about the distributions of the intensities, 
                        number of peptides remove after normalization, etc.
  --qc_report           PDF file to store multiple QC images (default: "TPA-QCprofile.pdf")
  --help                Show this message and exit.
```

#### 1. Calculate the TPA Value
The OpenMS tool was used to calculate the theoretical molecular mass of each protein. Similar to the calculation of IBAQ, the TPA value of protein-group was the sum of its intensity divided by the sum of the theoretical molecular mass.

#### 2. Calculate the Cellular Protein Copy Number and Concentration
The protein copy calculation follows the following formula:
```
protein copies per cell = protein MS-signal *  (avogadro / molecular mass) * (DNA mass / histone MS-signal)
```
For cellular protein copy number calculation, the uniprot accession of histones were obtained from species first, and the molecular mass of DNA was calculated. Then the dataframe was grouped according to different conditions, and the copy number, molar number and mass of proteins were calculated.

In the calculation of protein concentration, the volume is calculated according to the cell protein concentration first, and then the protein mass is divided by the volume to calculate the intracellular protein concentration.

### Datasets Merge - datasets_merge.py
There are batch effects in protein identification and quantitative results between different studies, which may be caused by differences in experimental techniques, conditional methods, data analysis, etc. Here we provide a method to apply batch effect correction.  First to impute ibaq data, then remove outliers using `hdbscan`, and apply batch effect correction using `pycombat`.


```asciidoc
python datasets_merge.py datasets_merge --data_folder ../ibaqpy_test/ --output datasets-merge.csv --verbose
``` 

```asciidoc
python datasets_merge.py --help
Usage: datasets_merge.py [OPTIONS]

  Merge ibaq results from compute_ibaq.py.

  :param data_folder: Data dolfer contains SDRFs and ibaq CSVs.
  :param output: Output file after batch effect removal.
  :param covariate: Indicators included in covariate consideration when datasets are merged.
  :param organism: Organism to keep in input data.
  :param covariate_to_keep: Keep tissue parts from metadata, e.g. 'LV,RV,LA,RA'.
  :param non_missing_percent_to_keep: non-missing values in each group.
  :param n_components: Number of principal components to be computed.
  :param min_cluster: The minimum size of clusters.
  :param min_sample_num: The minimum number of samples in a neighborhood for a point to be considered as a core point.
  :param n_iter: Number of iterations to be performed.
  :param verbose/quiet: Output debug information.

Options:
  Options:
  -d, --data_folder TEXT          Data dolfer contains SDRFs and ibaq CSVs. [required]
  -o, --output TEXT               Output file after batch effect removal. [required]
  -c, --covariate TEXT            Indicators included in covariate consideration when datasets are merged.
  --organism TEXT                 Organism to keep in input data.
  --covariate_to_keep TEXT        Keep tissue parts from metadata, e.g. 'LV,RV,LA,RA'.
  --non_missing_percent_to_keep FLOAT
                                  non-missing values in each group.
  --n_components TEXT             Number of principal components to be computed.
  --min_cluster TEXT              The minimum size of clusters.
  --min_sample_num TEXT           The minimum number of samples in a neighborhood for a point to be considered as a core point.
  --n_iter TEXT                   Number of iterations to be performed.
  -v, --verbose / -q, --quiet     Output debug information.
  --help                          Show this message and exit.
```

#### 1. Impute Missing Values
Remove proteins missing more than 30% of all samples. Users can keep tissue parts interested, and data will be converted to a expression matrix (samples as columns, proteins as rows), then missing values will be imputed with `KNNImputer`. 

#### 2. Remove Outliers
When outliers are removed, multiple hierarchical clustering is performed using `hdbscan.HDBSCAN`, where outliers are labeled -1 in the PCA plot. When clustering is performed, the default cluster minimum value and the minimum number of neighbors of the core point are the minimum number of samples in all datasets.

*Users can skip this step and do outliers removal manually.*

#### 3. Batch Effect Correction
Using `pycombat` for batch effect correction, and batch is set to `datasets` (refers specifically to PXD ids) and the covariate should be `tissue_part`.

### Citation

Wang H, Dai C, Pfeuffer J, Sachsenberg T, Sanchez A, Bai M, Perez-Riverol Y. Tissue-based absolute quantification using large-scale TMT and LFQ experiments. Proteomics. 2023 Oct;23(20):e2300188. doi: [10.1002/pmic.202300188](https://analyticalsciencejournals.onlinelibrary.wiley.com/doi/10.1002/pmic.202300188). Epub 2023 Jul 24. PMID: 37488995.

### Credits 

- Julianus Pfeuffer
- Yasset Perez-Riverol
- Hong Wang
