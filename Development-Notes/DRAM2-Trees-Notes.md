# DRAM2 Trees development notes and background


## DRAM2 Trees background and motivation

The purpose of DRAM2 Trees is to resolve differences in genes which have identical annotations. This is not to say these annotations are incorrect, it is to say that the annotation databases are not able to resolve the annotations to the level a user may need to state certain things about the genes they have annotated. 

Example: KEGG has a single KO for the genes: narG, narZ, nxrA. This results in the user not knowing which of these genes is actually present within their samples. However, researchers have been able to build phylogenetic trees with sequences of known subunits related to nitrogen reductase. Thus, by placing these sequences, which have identical KEGG KOs, on a tree with know sequences, one can decipher which sequences is which which subunit. 

DRAM2 Trees is being developed to do exactly this: identify pre-determined gene identifiers, obtained through annotation via the various databases, and place the corresponding sequences of these called genes on a pre-build phylogenetic tree. Then, based on either a closest neighbor metric or a pre-determined distance metric, assign a correct annotation to this called gene and update the annotations TSV file.

DRAM2 Trees will have pre-built trees for the following types of genes: nar/nxr, dsr, mcra, pmoa/amoa, and hydrogenases.

--------

## DRAM2 Trees development

### Current code status and how Trees is designed to be run

DRAM2 Trees should run if traits/adjectives is going to be run. This is due to the fact that the rules which assign traits/adjectives to input samples depends on the corrected annotations for the specified genes (nar/nxr, dsr, mcra, pmoa/amoa, and hydrogenases). However, depending on the computational demand, and time demand, to run these various trees, it may be beneficial to NOT run DRAM2 Trees in the event that user does not want to run traits/adjectives. Either way, the GitHub documentation should clearly state if the annotations the user gets have been corrected using phylogenetic trees.

The current state of DRAM2 Trees is only for the dmso tree and relies only on a nearest neighbor decision for calling the gene a given annotation. Then, DRAM2 parses the annotations TSV file and generates a new column named "tree_verified" 

As of now, it is one the user to provide the individual trees they want use to correct the annotations. First the user needs to specify they want to run the trees using the command line option: `--trees`. Then, the user may specify the individual trees with `--trees_list`. However, the end goal would be to hard-code `--trees_list` to include all of the various trees.

The TREES() process loops through `--trees_list` and corrects the annotations TSV for each tree that is ran. 

--------

### How to prepare the REFPKG

1) Obtain FASTA sequences

2) use muscle to align the sequences:

`grep -c ">" dmso_fortree.fasta  ## there are 3830 seqs`

`muscle -in dmso_fortree.fasta -out dmso_fortree_aligned.fasta`

3) protpipeliner.py to run raxml and find optimal sub model

`protpipeliner.py -i dmso_fortree_aligned.fasta -t 10 -b 100 -m low -a T`

4) Determine the number of positions using seqkit:

`seqkit stats dmso_refs.fasta.al`

```bash
#Output:
#file                format  type     num_seqs  sum_len  min_len  avg_len  max_len
#dmso_refs.fasta.al  FASTA   Protein        86  160,046    1,861    1,861    1,861
```

5) Create the `.refpkg` using taxit:

`taxit create -P dmso_package -l 1861 -p dmso_refs.fasta.model -f RAxML_info.dmso_refs.fasta --stats-type RAxML`

Additional notes:

It is best to ensure the created JSON files align with the already build `dmso` tree package. 

--------

#### Require REFPKG files and folders

Individual tree directories must reside within the `./assets/trees/` directory. The names of these directories, as well as the subsequent names of the contained files, must have the same naming scheme.

We will use the `dmso` tree as an example for the file structure.

Here are the contents of the `dmso/dmso.refpkg/` directory:
```bash
 dmso.refpkg/bipartitionsBranchLabels.dmso_refs.fasta_mode_low.renamed
 dmso.refpkg/CONTENTS.json
 dmso.refpkg/dmso.aln
 dmso.refpkg/dmso_phylo_model.json
 dmso.refpkg/dmso_refs.fasta
 dmso.refpkg/dmso_refs.fasta.model
 dmso.refpkg/dmso_search_terms.txt
 dmso.refpkg/dmso-tree-mapping.tsv
 dmso.refpkg/RAxML_info.dmso_refs.fasta
```

Explanation of each file:

**`dmso.refpkg/bipartitionsBranchLabels.dmso_refs.fasta_mode_low.renamed`**

Contains the branch-labled tree from RAxML.

**`dmso.refpkg/CONTENTS.json`**

This is the main JSON file describing the metadata, files, checksums and a minimal log. Importantly, this file needs to contain the correct names of all included files as well as the number of loci. This file is generated by `taxit` HOWEVER, some manual curation is needed (e.g. checksums). While the naming scheme can technically vary, it is suggested to keep a minimal and consistent naming scheme. Additionally, the current DRAM2 TREES code relies on this naming scheme. 

**`dmso.refpkg/dmso.aln`**

Contains the alignment file. 

**`dmso.refpkg/dmso_phylo_model.json`**

Contains information regarding the phylogenetic tree. For example here is the dmso file:

```bash
{
    "empirical_frequencies": true, 
    "datatype": "AA", 
    "subs_model": "LG", 
    "program": "RAxML version 8.2.9", 
    "ras_model": "gamma", 
    "gamma": {
        "alpha": 2.776473, 
        "n_cats": 4
    }
}
```

The information contained here can be found within the RAxML log file.

**`dmso.refpkg/dmso_refs.fasta`**

Contains the sequences, in FASTA format, which were used to generate the alignment and subsequent phylogenetic tree.

**`dmso.refpkg/dmso_refs.fasta.model`**

Output log file from `protpipeliner-pplacer.py` from ProtTest 3.2.

**`dmso.refpkg/dmso_search_terms.txt`**

Contains a single-column list of search terms. These can be KOs, EC numbers or strings. These are search for rigorously within the annotations TSV in order to select query_id values to place on a tree (their associated called gene sequences).

**`dmso.refpkg/dmso-tree-mapping.tsv`**

Tab-separated file containing the two required columns, `gene` and `call` respectively. `gene` contains the names associated with the gene sequences used to generate the tree and these must match with respect to the other files which reference them. `call` column contains the desired call. That is, if the correct gene call associated with this `gene` value is NxrA, then `NxrA` should dbe placed in the corresponding `call` row. The user may include additional columns for their personal reference but, these are not used within the DRAM2 TREES implementation. 

Example from the `dmso` tree:

```bash
gene	                                call	notes
A_Nitrobacter_hamburgensis_YP_578638  	NxrA	Cytoplasmic Nitrite oxidoreductase
Acidovorax delafieldii_NARG	        NarG	Nitrogen reductase
```

**`dmso.refpkg/RAxML_info.dmso_refs.fasta`**

Contains the RAxML log file used to populate the `dmso.refpkg/dmso_phylo_model.json` file.

--------

### Future development notes

It is suggested to determine if placing sequences on pre-built trees is less computationally expensive that building custom HMM databases with which to annotate from. 



