# ibaqpy

[![Python application](https://github.com/bigbio/ibaqpy/actions/workflows/python-app.yml/badge.svg)](https://github.com/bigbio/ibaqpy/actions/workflows/python-app.yml)

iBAQ (intensity Based Absolute Quantification) determines the abundance of a protein by dividing the total precursor intensities by the number of theoretically observable peptides of the protein. **ibaqpy** compute IBAQ values for proteins starting from a msstats input file and a SDRF experimental design file. This package provides multiple tools: 

- `peptide_file_generation.py`: generate a peptide file from a msstats input file and a SDRF experimental design file. 

- `peptide_normalization.py`: Normalize the input output file from script `peptide_file_generation.py`. It includes multiple steps such as peptidoform normalization, peptidorom to peptide summarization, peptide intensity normalization, and imputation. 

- `compute_ibaq.py`: Compute IBAQ values from the output file from script `peptide_normalization.py`.

### Collecting intensity files 

Absolute quantification files has been store in the following url: 

```
https://ftp.pride.ebi.ac.uk/pub/databases/pride/resources/proteomes/absolute-expression/
```

Inside each project reanalysis folder, the folder proteomicslfq contains the msstats input file with the structure `{Name of the project}_msstats_in.csv`. 

E.g. http://ftp.pride.ebi.ac.uk/pub/databases/pride/resources/proteomes/absolute-expression/PXD000561/proteomicslfq/PXD000561.sdrf_openms_design_msstats_in.csv 

### Generate Peptidoform Intesity file - peptide_file_generation.py

```asciidoc
python  peptide_file_generation.py --msstats PXD000561.sdrf_openms_design_msstats_in.csv --sdrf PXD000561.sdrf.tsv --output PXD000561-peptides.csv
```

The command provides an additional `flag` for compression data analysis where the msstats and sdrf files are compressed. 

```asciidoc
python peptide_file_generation.py --help
Usage: peptide_file_generation.py [OPTIONS]

  Conversion of peptide intensity information into a file that containers
  peptide intensities but also the metadata around the conditions, the
  retention time, charge states, etc.

  :param msstats: MsStats file import generated by quantms :param sdrf: SDRF
  file import generated by quantms :param compress: Read all files compress
  :param output: Peptide intensity file including other all properties for
  normalization

Options:
  -m, --msstats TEXT  MsStats file import generated by quantms  [required]
  -s, --sdrf TEXT     SDRF file import generated by quantms  [required]
  --compress          Read all files compress
  -o, --output TEXT   Peptide intensity file including other all properties
                      for normalization
  --help              Show this message and exit.
```

### Peptide Normalization - peptide_normalization.py

Peptide normalization starts from the output file from script `peptide_file_generation.py`. The structure of the input contains the following columns: 

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
- SampleID: Sample ID `(e.g. PXD000561-Sample-54)`
- StudyID: Study ID `(e.g. PXD000561)`. In most of the cases the study ID is the same as the ProteomeXchange ID.

#### Removing Contaminants and Decoys

The first step is to remove contaminants and decoys from the input file. The script `peptide_normalization.py` provides a parameter `--contaminants` for the user to provide a file with a list of protein accessions which represent each contaminant in the file. An example file can be seen in `data/contaminants.txt`. In addition to all the proteins accessions, the tool remove all the proteins with the following prefixes: `CONTAMINANT` and `DECOY`.

#### Peptidoform Normalization

A peptidoform is a combination of a `PeptideSequence(Modifications) + Charge + BioReplicate + Fraction`. In the current version of the file, each row correspond to one peptidoform. 

The current version of the tool uses the parackage [qnorm](https://pypi.org/project/qnorm/) to normalize the intensities for each peptidofrom. **qnorm** implements a quantile normalization method. 

#### Peptidoform to Peptide Summarization

For each peptidoform a peptide sequence (canonical) with not modification is generated. The intensity of all peptides group by biological replicate are `sum`. 

Then, the intensities of the peptides across different biological replicates are summarize using the function `median`. 

At the end of this step, for each peptide, the corresponding protein + the intensity of the peptide is stored. 

#### Peptide Intensity Imputation and Normalization

Before the final two steps (peptide normalization and imputation), the algorithm removes all peptides that are source of missing values significantly. The algorithm removes all peptides that have more than 80% of missing values and peptides that do not appear in more than 1 sample. 

Finally, two extra steps are performed: 

- ``peptide intensity imputation``: Imputation is performed using the package [missingpy](https://pypi.org/project/missingpy/). The algorithm uses a Random Forest algorithm to perform the imputation.
- ``peptide intensity normalization``: Similar to the normalization of the peptidoform intensities, the peptide intensities are normalized using the package [qnorm](https://pypi.org/project/qnorm/).

### Compute IBAQ



### Credits 

- Julianus Pfeuffer
- Yasset Perez-Riverol 
