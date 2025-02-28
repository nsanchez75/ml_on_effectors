# AlphaFold Tests

## What AlphaFold2 Produces

TODO

## Papers to Cite

- Jumper, J et al. Highly accurate protein structure prediction with AlphaFold. Nature (2021).
- Varadi, M et al. AlphaFold Protein Structure Database: massively expanding the structural coverage of protein-sequence space with high-accuracy models. Nucleic Acids Research (2021).

## Useful Sources

- [Using AF in ChimeraX](https://www.youtube.com/watch?v=gIbCAcMDM7E&authuser=1)

## Experiments: Data and Results

### Figuring Out How to Gather Data from AlphaFold

(10/25/2023)

- ran AlphaFold2 Colab file on test sequences (found in data) similar to how I had run the data on ESMFold
  - made sure to omit the '*' at the end of the sequences
  - didn't touch anything in the Colab (just did 'Run All' like it says to do in the Colab)
- results found in 10/25/2023

#### Exploring Google Cloud and BigQuery

(11/1/2023)

- found some sequences in Uniprot that come from Bremia Lactucae. Apparently, Alphafold provides predicted structures for the sequences. Link: <https://www.uniprot.org/uniprotkb?query=(taxonomy_id:4779)>
- I created the fasta file `uniprotkb_taxonomy_id_4779_2023_11_01.fasta` by downloading a fasta file from the website above. The file lists all of the Bremia lactucae sequences found in the Uniprot database.
- I will use this to extract all of the Uniprot accession IDs so that I can download all of the AlphaFold predictions associated with Bremia lactucae.
- The predictions may be found in [this dataset on Google Cloud](https://console.cloud.google.com/storage/browser/public-datasets-deepmind-alphafold-v4;tab=objects?prefix=&forceOnObjectsSortingFiltering=false)

(11/2/2023)

I created a Google Cloud account and explored methods to get the AlphaFold sequences from the database. So far, this is what I've found:

- [Bremia lactucae's taxonomy ID is 4779](https://www.ncbi.nlm.nih.gov/Taxonomy/Browser/wwwtax.cgi?id=4779)
- The list of sequences can be found in the BigQuery metadata of the database
- You need to get a Google Cloud account in order to access BigQuery (means inputting credit/debit card info)
- **MAKE SURE YOU KNOW WHAT YOU'RE DOING W/ GOOGLE CLOUD**
  - costs can sneak up on you sometimes so make sure to not activate your account (permits payments to begin after the free trial) and, if you do need to activate it, set up budget alerts

(11/3/2023)

I used these steps provided by the [AlphaFold DB Github README.md repo](https://github.com/google-deepmind/alphafold/blob/main/afdb/README.md). So far, I successfully downloaded gsutil using [this guide](https://cloud.google.com/storage/docs/gsutil_install) and tried to make it work, but I need to figure out how to input my credentials as the provided gsutil code (`gsutil -m cp gs://public-datasets-deepmind-alphafold-v4/proteomes/proteome-tax_id-[TAX ID]-*_v4.tar .`) doesn't work.

I'll start looking more into the authentication stuff while also figuring out if I can access the database in other ways (ex: use the Uniprot list file).

Here is a screenshot of my SQL script in BigQuery where I managed to find all of the Bremia-lactucae AF results:

![Screenshot of BigQuery SQL script into metadata](notebook-images/image.png)

#### Getting Data Through Kelsey Instead

I downloaded the `alphafoldbulk.py` script that may help me download all of the sequences. Kelsey also sent me a bash script that is much simpler:

```bash
file=$1

while read -r line
do
  wget  "https://alphafold.ebi.ac.uk/files/"$line"-model_v4.pdb"
done <$file
```

She also gathered all of the data, so I will look into her directory later and grab all of the data. I might change this code later if I find a better way to specifically grab the Bremia lactucae related pdf files. For now, I will just grab the data she had gathered. I kept the data in the git-untracked `AF_db_data` directory in the lab server (kept it there because it is very large).

(11/6/2023)

I created a script that detects if the data produced by Kelsey had all the entry IDs extracted from BigQuery. It seems like some of the entry IDs were not found.

**TODO:** figure out what happened to these missing entry IDs.

### Linking ORFs for B_lac-SF5 and AF predicted sequences thru BLAST (11/6/2023 --> updated 11/15/2023)

(11/6/2023 --> updated 11/15/2023)

Here is the link for grabbing the [AlphaFold metadata from BigQuery](https://console.cloud.google.com/bigquery?p=bigquery-public-data&d=deepmind_alphafold&page=dataset&project=woven-precept-403918&ws=!1m4!1m3!3m2!1sbigquery-public-data!2sdeepmind_alphafold).

#### Gathering Data

I created a SQL script in BigQuery that creates the protein sequences that relate to an AF entry ID. The script is below. The output of this SQL entry is `af_entry-ids_uniprot-seqs_for_blac-sf5.csv`. To fact-check that the correct csv file is being used, remember that the number of Bremia lactucae sequences in uniprot is 8951 (remember to check lines to ensure the right files are being used).

 After analyzing the output of the resulting csv file, I found that the entryId was the same as the uniprotId with an `AF-` at the front and other stuff after it. To use the csv file to produce the BLASTp result, I converted the csv into a tsv file called `af-uniprot-id_uniprot-seq.no_headers.tsv` that only has the AF IDs and uniprot sequences from the csv file. Then, I produced a FASTA file called `af-uniprot-id_uniprot-seq.fasta` using the Python script `convert_tsv_to_fasta.py`. Here is the SQL script that produced the csv file:

```SQL
select entryId, uniprotId, uniprotSequence from `bigquery-public-data.deepmind_alphafold.metadata`
where taxId = 4779
```

Kelsey gave me this script that will help determine which sequences are related to each other via BLASTp:

```bash
blastp -query [query.fasta] -db [db.fasta] -evalue 1e-10 -outfmt "6 std qcovs" -out [name]

# database = all Bremia lactucae ORFs
# query=uniprot sequences with alphafold

# output = table of best hits
```

I used `af-uniprot-id_uniprot-seq.fasta` as my query fasta file and `B_lac-SF5.protein.fasta` as my database fasta file. In order to use the database fasta file, I need to convert it into a BLASTp database. To do this, I am using the `makeblastdb` command. Information on this command can be found on the [NIH website](https://www.ncbi.nlm.nih.gov/books/NBK569841/). The website example uses a taxId map text file; however, it may be difficult to do so.

This is the error code that ran when I didn't create a database from the database FASTA file first:

```text
BLAST Database error: No alias or index file found for protein database [B_lac-SF5.protein.fasta] in search path [/share/rwmwork/nsanc/kelsey_work/ml-on-effectors/alphafold-tests/data/2023_11_1-6::]
```

This is the command I ran in order to make the database (I ended up having to run this again on 11/15/2023 in the date's data directory):

```bash
makeblastdb -in B_lac-SF5.protein.fasta -title "Bremia Lactucae ORF Sequences" -dbtype prot > B_lac-SF5_db_creation.log

## INFO:
# -in           : input FASTA file
# -title        : BLAST database title
# -dbtype prot  : makes database a protein type
```

The status content is logged in `B_lac-SF5_db_creation.log` (remember to set the output of makeblastdb to a .log next time), Here's what it can look like:

```log
Building a new DB, current time: 11/06/2023 18:03:03
New DB name:   /share/rwmwork/nsanc/kelsey_work/ml-on-effectors/alphafold-tests/data/2023_11_1-6/B_lac-SF5.protein.fasta
New DB title:  Bremia Lactucae ORF Sequences
Sequence type: Protein
Keep Linkouts: T
Keep MBits: T
Maximum file size: 1000000000B
Adding sequences from FASTA; added 105272 sequences in 7.52235 seconds.
```

I am running a script I called `blastp_q_on_ORFs.sh` where the inputs are:

- $1: input query FASTA file
- $2: input database FASTA file
- $3: output filename

and the script runs BLASTp. Here is the command:

```bash
blastp -query af-uniprot-id_uniprot-seq.fasta -db /share/rwmwork/nsanc/kelsey_work/ml-on-effectors/alphafold-tests/data/2023_11_1-6/B_lac-SF5.protein.fasta -evalue 1e-10 -outfmt "6 std qcovs" -out blastp_output.txt
```

And here is the script (as I had used on 11/15/2023 in the date's data directory):

```bash
./blastp_q_on_ORFs.sh af-uniprot-id_uniprot-seq.fasta /share/rwmwork/nsanc/kelsey_work/ml-on-effectors/alphafold-tests/data/2023_11_15/B_lac-SF5.protein.fasta blastp_output_db_B_lac-SF5_q_blac-uniprot.txt
```

#### Results

Here is a 3-line sample of what the file ends up looking like. By the way, for clarity, I renamed `blastp_output.txt` to `blastp_output_db_B_lac-SF5_q_blac-uniprot.txt` (update 11/15/2023: I just made the output file name as this which is more efficient anyways). I referenced what the columns are through [this website about BLAST output format 6](https://pascal-martin.netlify.app/post/blastn-output-format-6/). Just a warning, it takes a decent amount of time for blastp to produce a result so I would recommend (if running on its own rather than through a pipeline) to run it in the background or dedicate a screen to it.

### Analyzing Predited AF Sequences w/ EffectorO Prediction, RXLR-EER, WY, CRN, SP, etc

#### Linking Uniprot IDs and ORFs (deprecated)

(11/8/2023 --> deprecated w/ 11/13/2023 stuff)

To analyze the AlphaFold sequences, I am creating a table that links Uniprot Accession IDs to their ORF sequence counterparts. For this, I am going to use the `awk` command on the file `blastp_output_db_B_lac-SF5_q_blac-uniprot.txt` through the CLI. The result is in a file called `uniprot-id_to_ORF-blac-seq.txt`. Here is the code:

```bash
awk -F' ' '{print $1"\t"$2}' blastp_output_db_B_lac-SF5_q_blac-uniprot.txt > uniprot-id_to_ORF-blac-seq.txt
```

After creating the table, I want to make sure that each uniprot ID has a unique associated ORF. Here is the code I wrote to do this:

```bash
awk -F' ' '{print $2}' uniprot-id_to_ORF-blac-seq.txt | uniq -c | sort | grep -v -e 1 > count_num_duplicate_ORFs.txt
```

I found that there are many sequences that have more than one associated uniprot ID.

To extract the uniprot IDs associated with each ORF, I am developing a bash script called `identify_duplicate_ORFs.sh` that will go through `blastp_output_db_B_lac-SF5_q_blac-uniprot.txt`, extract the lines associated with any duplicate ORF, and create a new blastp file that does not contain any duplicate ORFs. The resulting outputs will be called `blastp_output_db_B_lac-SF5_q_blac-uniprot.duplicate_uniprot-ids.txt` and `blastp_output_db_B_lac-SF5_q_blac-uniprot.unique_uniprot-ids.txt` to contain duplicate and unique ORF lines respectively. Here are the input parameters for the bash script:

- $1: input BLASTp result
- $2: input list of duplicate ORFs
- $3: output duplicate ORFs filename
- $4: output unique ORFs filename

This is how I am running the script:

```bash
# removing count column from duplicate ORFs file
cat count_num_duplicate_ORFs.txt | awk -F' ' '{print $2}' > duplicate_ORFs.txt

./identify_duplicate_ORFs.sh blastp_output_db_B_lac-SF5_q_blac-uniprot.txt duplicate_ORFs.txt blastp_output_db_B_lac-SF5_q_blac-uniprot.duplicate_uniprot-ids.txt blastp_output_db_B_lac-SF5_q_blac-uniprot.unique_uniprot-ids.txt
```

note: `uniprot-id_to_ORF-blac-seq.removed_duplicates.txt` will have a line at the beginning like this:

```text
### blastp output with only duplicate ORFs
```

If this file is read later, make sure to deal with this line.

#### Creating the Table

To create the table, I am going to use the `pandas` module in Python. For this, I am creating a conda environment meant to be used for all projects on the ml-on-effectors repo. The `environment.yml` will be placed at the root of the repo. I want to use the latest Python because I feel like it, so the environment will run on Python 3.12 (hopefully this doesn't run into any issues in the future).

(11/9/2023)

I am creating a Python script called `create_uniprot-id_table.py` that determines if a uniprot ID is associated with:

- ORFs with >= 0.85 effector prediction
- WY
- RXLR
- CRN
- SP
- and more when I evaluate other classifications

In order to create an ORF list with the >=0.85 predicted effectors, I ran this command on the FASTA file associated with it:

```bash
grep -Po '>(\w+)' predicted_blac-SF5_effectors_filtered_L0.85.fasta | cut -c 2- > predicted_blac-SF5_effectors_filtered_L0.85.list_of_ORFs.fasta
```

I will be using the whole BLASTp output rather than just the uniprot IDs with unique ORFs first. Here is the command I will be using:

```bash
python3 create_uniprot-id_table.py uniprot-id_to_ORF-blac-seq.txt RXLR_and_EER IDs_rxlr_and_eer.tab CRNs blac-sf5-v8-crns.txt SP sp_B_lac-SF5.filterlist.txt WY_Domain wy_cleaved_sp_B_lac-SF5.protein.fasta ov0.85_predicted predicted_blac-SF5_effectors_filtered_L0.85.list_of_ORFs.fasta
```

(11/13/2023)

Kelsey said that there is a better way to filter the BLASTp output to give the best possible hit. Here is the code she provided in addition to this:

```R
WYs=read.table("results_allWYs_with_names.tab", sep="\t", stringsAsFactors=F, comment.char="", header=F)
names(WYs) = c("qseqid", "sseqid", "pident", "length", "mismatch", "gapopen", "qstart", "qend", "sstart", "send", "evalue", "bitscore","qcovs")
#sort by percent identity so you can get the best hit for each gene
order_WYs = WYs[order(-WYs$pident),]
#remove duplicates - this keeps the first instance which will be higher since they were sorted
uniq_hpw = order_WYs[!duplicated(order_WYs$qseqid),]
```

I rewrote some names of variables in the code for generalization and also allowed for the user to input their own BLASTp file (requires `optparse`).

Also, based on Kelsey's recommendation, I changed the Python script `create_uniprot-id_table.py` so that there are no spaces in column names. This only affected "Uniprot ID" and "ORF Sequence" (converted to "Uniprot_ID" and "ORF_Sequence" respectively).

(11/14/2023)

I edited the R script given to me by Kelsey (now called `tabularize_blastp_output.r`) to produce a table that provides the best hits for all the AF uniprot sequences. The following command was used:

```bash
Rscript tabularize_blastp_output.r -i blastp_output_db_B_lac-SF5_q_blac-uniprot.txt
```

The resulting output file from this code is called `blastp_output_db_B_lac-SF5_q_blac-uniprot.filtered_best_hits.txt`.

note: I ran the following code to double check if the ORF IDs were truly unique in the new file and it ended up being true:

```bash
cat blastp_output_db_B_lac-SF5_q_blac-uniprot.filtered_best_hits.txt | tail -n +2 | awk -F'\t' '{print $2}' | uniq -c | less
```

----

(11/29/2023)

There are some missing AF IDs in the filtered BLASTp output. According to `af_entry-ids_for_blac-sf5.csv`, there are 8952 entries; however, there are 7973 entries in `blastp_output_db_B_lac-SF5_q_blac-uniprot.filtered_best_hits.txt`. I ran this in the CLI to determine what AF IDs were missing:

```bash
diff <(cat af_entry-ids_for_blac-sf5.csv | tail -n +2 | sort) <(awk -F'\t' '{print $1}' blastp_output_db_B_lac-SF5_q_blac-uniprot.filtered_best_hits.txt | tail -n +2 | sort) > missing_AF_IDs.temp.txt
grep -Po '< (\w+-\w+)' missing_AF_IDs.temp.txt | cut -c 3- > missing_AF_IDs.txt
rm missing_AF_IDs.temp.txt
```

I am going to test the e-value of these missing AF IDs on our Bremia lactucae SF5 group. First, I created the filtered AF fasta file:

```bash
./filter_fasta_with_list.sh af-uniprot-id_uniprot-seq.fasta missing_AF_IDs.txt
```

This created `missing_AF_IDs.filtered.fasta`.

Next, I created a database of B_lac-SF5.protein.fasta in the data/2023_11_29 directory (this line of code was copied from line 116):

```bash
makeblastdb -in B_lac-SF5.protein.fasta -title "Bremia Lactucae ORF Sequences" -dbtype prot > B_lac-SF5_db_creation.log
```

Next, I ran BLASTp on `missing_AF_IDs.filtered.fasta` without a specified e-value.

```bash
blastp -query missing_AF_IDs.filtered.fasta -db /share/rwmwork/nsanc/kelsey_work/ml-on-effectors/alphafold-tests/data/2023_11_29/B_lac-SF5.protein.fasta -outfmt "6 std qcovs" -out blastp_missing_AF-IDs_output.txt
```

After looking at their e-values, I found that their scores are above the previosly specified 1e-10 cutoff. Here is the code and 10-line output displaying the e-value scores in ascending order:

```bash
awk -F'\t' '{print $11}' blastp_missing_AF-IDs_output.txt | sort -g | head -n 10

3e-17
3e-11
1e-10
1e-10
1e-10
1e-10
2e-10
2e-10
2e-10
2e-10

# anomalies
AF-A0A484DQ54-F1  Blac_SF5_v8_5_ORF85373_fr2  22.78 496 276 20  35  490 438 866 3e-17 85.192
AF-A0A484E4Y9-F1  Blac_SF5_v8_24_ORF337457_fr2  54.72 53  24  0 1 53  1 53  3e-11   55.190
```

The last two entries I wrote out seem to be anomalies since their scores are below the 1e-10 threshold. Maybe their pident was too low?

note: the -g command for `sort` sorts the lines numerically.

----

##### BLASTp on NCBI Genome + AF IDs

(11/29/2023)

I downloaded the Bremia lactucae SF5 protein dataset off of [the NCBI website](https://www.ncbi.nlm.nih.gov/datasets/genome/GCA_004359215.2/). The file is named `blac_SF5_ncbi_proteins.faa`.

This is the code that was used to create the database for ORFs + Gene Models (protein seq).

```bash
makeblastdb -in blac_SF5_ncbi_proteins.faa -title "Bremia Lactucae NCBI Protein Sequences" -dbtype prot > B_lac-SF5_ncbi_proteins_db_creation.log
```

And I used this code to BLASTp the NCBI proteins and the AF ID fasta file (note that this BLASTp uses the default e-value=10):

```bash
blastp -query af-uniprot-id_uniprot-seq.fasta -db blac_SF5_ncbi_proteins.faa -outfmt "6 std qcovs" -out blastp_output_AF-ID_ncbi-proteins.txt
```

And I found out that this stuff was kinda redundant. The filtered BLASTp output shows a bunch of 100% pidents and qcovs. Very nice.

----

I also created a new version of `create_uniprot-id_table.py` called `create_ORF-id_table.py` to represent the new table structure that Kelsey wants. In this new structure, the table headers are ordered as follows:

|ORF_sequence_ID|RXLR_EER|CRN_motif|SP|WY_Domain|effectorO_score|best_blast_hit_AF_ID|pident|length|mismatch|gapopen|qstart|qend|sstart|send|evalue|bitscore|qcovs|
|---------------|--------|---------|--|---------|---------------|--------------------|------|------|--------|-------|------|----|------|----|------|--------|-----|
|...............|........|.........|..|.........|...............|....................|......|......|........|.......|......|....|......|....|......|........|.....|

(11/15/2023 - 11/16/2023)

I realized that I used the wrong AF sequence ID fasta file! I redid my process from 11/6/2023 (see above). I am running everything up to the stuff previosly and then talk to Kelsey about these results (should be pretty fast since I've got the general pipeline down). Here's the code that I ran for `create_ORF-ID_table.py`. As of now the code doesn't include RXLR-EER since there needs to be some analysis still, but I'll deal with it tomorrow.

(update 11/20/2023: added `IDs_rxlr_and_eer.tab` to script below)

```bash
python3 create_ORF-ID_table.py -i blastp_output_db_B_lac-SF5_q_blac-uniprot.filtered_best_hits.txt -f wy_cleaved_sp_B_lac-SF5.protein.fasta sp_B_lac-SF5.filterlist.txt blac-sf5-v8-crns.txt predicted_blac-SF5_effectors_filtered_L0.85.list_of_ORFs.fasta IDs_rxlr_and_eer.tab
```

The resulting TSV file is named `blastp_output_db_B_lac-SF5_q_blac-uniprot_on_WY-Domain_SP_CRN-motif_predicted-effectors-ov-85.tsv`.

Just for personal reference, there are many ways to get the data from this TSV file. Here are some command examples:

- determining what AF-uniprot IDs could be associated w/ ORF classifications

```bash
cat blastp_output_db_B_lac-SF5_q_blac-uniprot_on_WY-Domain_SP_CRN-motif_predicted-effectors-ov-85.tsv | awk -F'\t' '{print $1 "\t" $14 "\t" $15 "\t" $16}' | less
```

- checking what AF-uniprot IDs are associated w/ WY Domains

```bash
cat q_blac-uniprot_on_WY-Domain_SP_CRN-motif_predicted-effectors-ov-85.tsv | awk -F'\t' '$14=="1" {print $1 "\t" $14 "\t" $15 "\t" $16}' | less
```

Also, I've updated all directories that have single-digit numbers in them to add a '0' in order to sort out their ordering.

#### Summarizing and Analyzing Table Data

**How many effectors of each class have AF structures in the database?** To answer this question, I first need to determine the best method of displaying the data for this. Here are some suggestions:

- Summary Table (from Kelsey)
- Venn Diagram for basic numerical data (can distinguish how many are identified w/ multiple classes + no class)

<!--
Here is a script produced by ChatGPT that might be able to construct a Venn Diagram that can list the number of AF IDs not in any of the sets (all the classifications)

```R
# Install and load the VennDiagram package
install.packages("VennDiagram")
library(VennDiagram)

# Create example data
set1 <- c("A", "B", "C", "D")
set2 <- c("C", "D", "E", "F")
set3 <- c("G", "H", "I", "J")

# Create the Venn diagram
venn.plot <- venn.plot(list(Set1 = set1, Set2 = set2, Set3 = set3),
                       category.names = c("Set1", "Set2", "Set3"),
                       filename = NULL, output = TRUE)

# Add a number for data that isn't in any of the sets
venn.plot <- venn.plot + 
  draw.single(0.9, 0.9, "0", category = "Only in diagram")

# Display the Venn diagram
grid.draw(venn.plot)
```
-->

I decided to implement the summary table creation inside of the `create_ORF-ID_table.py` script. The resulting summary table will be given as a log file called [name of log file]. In order to produce the summary log with the different combinations of headers, I used the `itertools` module, particularly its `combinations` function (which doesn't require a pip install thankfully). THe resulting output file is called `blastp_output_db_B_lac-SF5_q_blac-uniprot_on_WY-Domain_SP_CRN-motif_predicted-effectors-ov-85.summary_table.log`.

(11/19/2023)

After a lot of complicated configuration for the past couple of days, I've managed to develop a summary table creator in the python script. It involves using the idea of combinations and reduction in order to perform its process. To test if the values are correct, I've been using awk commands. Below is an example awk command that checks if the last 4 columns produce the same count as the summary table

```bash
awk -F'\t' '$(NF-3) == 0 && $(NF-2) == 0 && $(NF-1) == 0 && $NF == 0 {print}' blastp_output_db_B_lac-SF5_q_blac-uniprot_on_WY-Domain_SP_CRN-motif_predicted-effectors-ov-85.tsv | wc -l
```

Here is a sample of the output summary log (this one didn't have RXLR but the other summary logs below do):

```text
Neither WY-Domain nor SP nor CRN-motif nor predicted-effectors-ov-85: 6374
predicted-effectors-ov-85: 536
CRN-motif: 0
WY-Domain: 0
SP: 943
predicted-effectors-ov-85 and CRN-motif: 0
predicted-effectors-ov-85 and WY-Domain: 0
predicted-effectors-ov-85 and SP: 52
CRN-motif and WY-Domain: 0
CRN-motif and SP: 21
WY-Domain and SP: 40
predicted-effectors-ov-85 and CRN-motif and WY-Domain: 0
predicted-effectors-ov-85 and CRN-motif and SP: 3
predicted-effectors-ov-85 and WY-Domain and SP: 3
CRN-motif and WY-Domain and SP: 0
predicted-effectors-ov-85 and CRN-motif and WY-Domain and SP: 0
```

(11/20/2023)

Kelsey says that the summary table should look more like this:

| No SP |  |
| ----- | - |
| Predicted non-effector (no RXLR, WY, CRN, EffO) | 6374 |
| Predicted effector (EffectorO > 0.85) | 484 |

| SP |  |
| ----- | - |
| Predicted non-effector (no RXLR, WY, CRN, EffO) | 943 |
| CRN-motif | 21 |
| WY-Domain | 40 |
| EffectorO (>0.85) | 52 |

| Total |
| ----- |
| 7914  |

This is because if the sequence has RXLR, WY, CRN, or anything known to be an effector, then it is bound to have a signal peptide as well. This is proven with the summary table I produced with the different combinations of all the headers; all of the known effectors only had a quantifiable count if SP was involved.

To make this, I commented out my previous summary table creator in `create_ORF-ID_table.py` and am creating a new Python script called `analyze_ORF-ID_table.py` that will produce the resulting summary table. *As a note for the future, I assume that the known effectors for non-SP data comes from effector predictions >0.85. Make sure to generalize this if necessary.*

```bash
python3 analyze_ORF-ID_table.py blastp_output_db_B_lac-SF5_q_blac-uniprot_on_WY-Domain_SP_CRN-motif_predicted-effectors-ov-85_RXLR-EER.tsv
```

**TODO:** Make sure to identified sequences that fulfill more than one class under SP (gonna use itertools again yay). This is because the resulting table ends up being:

(11/28/2023 update - added parentheses in text below)

```text
No SP:
        Predicted Non-Effectors: 6374
        Predicted Effectors: 536
SP:
        Predicted Non-Effectors: 916
        WY-Domain: 43 (12 w/ RXLR)
        CRN-motif: 24
        predicted-effectors-ov-85: 58 (try to specify ones that dont have WY + RXLR-EER)
        RXLR-EER: 50 (w/o WY)

Total: 8001
```

while the actual file `blastp_output_db_B_lac-SF5_q_blac-uniprot_on_WY-Domain_SP_CRN-motif_predicted-effectors-ov-85_RXLR-EER.tsv` has only 7973 lines (7972 excluding the header row).

(11/30/2023)

Here is the new table that Kelsey wants me to create:

```text
Non-secreted (SignalP)
  Predicted non-effector (no RXLR, WY, CRN, EffO)
  CRN-motif
  WY-Domain
    with RXLR-EER
  RXLR-EER (no WY)
  EffectorO (>0.85) (no other effector domains)
  
  Total non-secreted


Secreted (SignalP)
  Predicted non-effector (no RXLR, WY, CRN, EffO)
  CRN-motif
  WY-Domain
    with RXLR-EER
  RXLR-EER (no WY)
  EffectorO (>0.85) (no other effector domains)
  
  Total secreted


Total proteins in UniProt/AF DB
```

Here is the code I used to determine if there were no sequences that contain CRN ($16) and either WY ($14) or RXLR-EER ($18).

```bash
awk -F'\t' 'BEGIN{OFS="\t"} $14==1 && $16==1 || $18==1 && $16==1 {print}' blastp_output_db_B_lac-SF5_q_blac-uniprot_on_WY-Domain_SP_CRN-motif_predicted-effectors-ov-85_RXLR-EER.tsv | wc -l
```

note: '&&' has precedence over '||' according to [this StackOverflow post](https://stackoverflow.com/questions/42096063/multiple-conditions-of-and-and-or-in-one-awk-command).

**TODO:** turn this into a bash script for generalization

### Predicting Bremia Lactucae Sequences

#### Running on ColabFold

I ran this 971aa sequence on ColabFold to test how long it runs on long sequences (>=860aa):

```text
>Blac_SF5_v8_1_ORF2357_fr6 Blac_SF5_v8_1:1091517..1094432
MQASNSDIRRHFKQFDDELQDSDQTFSQSVQYSRPRHRPSSIKDASNNSVKPLRCRFPTSAFFSDSTDDDVSRPIATSPSRSQLSRSACNRSKYTKHYNVPLHMSVGPREGQIGKTHMPRLHVHPIQFPITRSVLWDRRPLSAGPIKLSLEQILLQFIVSADALKEIYKLYREKKDAQDNYIVLCYLYGNASMGIENKVLLKRVMRHKEDVRAAGIWCVPVLSMAITEDNSELLRNSYVSSVQSVQSSYRDEYTDDVSCKLQPKLLVFQSSPHAAQLEFQLECVASPVLFTFNLVRNLPLLMTPLAATLAKRDLTIRSGNLRSGYLTLDSTRKAVPLLKVDPLVMQYPIIGVWVYGVQIDDAWDDDRARRQLASPFLYFACIGFLMSEAIRERVGPAKNTFLVALYPANDLECVGTIGTLPRFFECSYSSFLSPPEEPLPLELYSHRRSCLVGVSKFSTDVEFILSAATTREWEEARHQMKIPASLRTEIDNLECKRSRLGAYKEQERSVALQDIRLVSVGDEDQVDGLSGWTITDDAIKKSVSLGLNGATPALKSKAENAAFANVNLVDDQENLLIGGQEKYEAKSDSDISKTSSSKNVEVATEAAKRVGSFYNSNKSRSCCKTQQLLTIQHQQILENQQKQLHDMQDQISQLRHLLSVARSGSSSERQTNASLDDGIDESGDNKLSVDAIHTHTSSAGCKPEDGNTCLQLSMASESQRNEDDSDSDINSLFKVENDRDDSMDFSLSSLNLSSIRSGSDAGLSSQSSSPVGRQAKQLCPLEMHDKDLAVEKAAHSDVNRSTRSEPPEVPMSNGETAQQLALSEICGDDEADSCKLQEHENAEERSALKLFLGELDEFGSSSNPSAVDNVENVHANIPNFVVDNRSRGDNESSGDKFSKIERLLSPDAYLRKIGGFVDLHKGCFTTPSLDFHSFCVPRIKYSTESPEYPMSDSEDEETRLIERKYKQLMAA*
```

and it took **~1 hour**. There are 899 860+ aa sequences, so it would take ~37.46 days to run all of the longer sequences.

To test the lower bound, I ran this 99aa sequence:

```text
>Blac_SF5_v8_1_ORF38_fr1 Blac_SF5_v8_1:1258..1554
MDIYRSHPATFVAAVTIASNAECGLKSSRVGMHGDNVSSLSTSGPSIKPQLTDISLAEEKEEPQAVKQYNNNRMLYVRKHKALVHGVCCPRCHVQHFR*
```

and it took **~1 minute**.

(11/27/2023)

I have discovered from some Google searching that there may be a way to run Google Colab notebooks locally; this would mean it could be possible for ColabFold to be run on the lab server. I found this [StackOverflow post](https://stackoverflow.com/questions/50194637/colaboratory-how-to-install-and-use-on-local-machine) that explains one possible way to install Google Colab onto the lab server. Here are the instructions:

- git clone [this repo](https://github.com/googlecolab/colabtools)
- when `cd`ed into the repo, run `python3 setup.py install`

I also looked for ColabFold repos in GitHub. I found these two:

- [ColabFold](https://github.com/sokrypton/ColabFold)
- [ColabFold for Local PC](https://github.com/YoshitakaMo/localcolabfold)

After looking more into this, I noticed that this GitHub repo may be what Kakawa is using to run ColabFold.

Now, it is time for me to determine what sequences are not present in the AF database. This is under the assumption that only the best hits of ORFs and AF sequences are the only ORFs present in the AF database. I created the list files used in the code below with grep -Po (same as in line 221) and the awk function on `B_lac-SF5.protein.fasta` and `blastp_output_db_B_lac-SF5_q_blac-uniprot.filtered_best_hits.txt` respectively. I used [this website](https://www.baeldung.com/linux/finding-unique-text-between-two-files#:~:text=Using%20grep,in%20one%20of%20the%20files.&text=We%20use%20the%20%2DF%20option,for%20specifying%20the%20pattern%20file.) to learn about the `-Fxvf` flag for grep.

```bash
grep -Fxvf blastp_output_db_B_lac-SF5_q_blac-uniprot.filtered_best_hits.ORFs.txt B_lac-SF5.protein.list.txt > blac_ORFS_not_in_AF-db.txt
```

(11/28/2023)

I am creating a filtered version of `B_lac-SF5.protein.fasta` by using a Shell script `filter_fasta_with_list.sh` that takes in the list file `blac_ORFS_not_in_AF-db.txt` and the fasta file to produce the newly filtered fasta file containing ORF sequences not found in the AF database.

**TODO:** continue with this

#### Running on Kakawa

(11/27/2023)

Kelsey is currently working on making this work.

#### Running on Local Computer

Kelsey talked about using local computers in order to make the computation time much faster. The GitHub link can be found on line 438 as well as through [this link](https://github.com/YoshitakaMo/localcolabfold).

Here is a [link](https://saturncloud.io/blog/how-to-run-google-colab-locally-a-step-by-step-guide/) that explains how to set up Colab locally.

#### Linking NCBI IDs to AF IDs

To do this, I downloaded an excel file called `Bremia-WY-NCBI-seqs.xlsx` that contains NCBI IDs of some protein sequences from Kelsey. Kelsey wanted me to map these sequences to their AF IDs. For this, I used [this website](https://www.uniprot.org/id-mapping) to first map the NCBI IDs to their Uniprot ID counterparts; this is because AF IDs use Uniprot IDs as a part of their naming scheme. This resulted in the fasta file `ncbi-id_to_unprot-id_2023_11_28.fasta`.

(12/6/2023-12/7/2023)

I moved the following files into the data directory 2023_12_06. I am focusing primarily on WY sequences first:

1. `Bremia-WY-NCBI-seqs.csv`: contains all of the WY-Domain sequences from NCBI
    - headers: NCBI_ID_SF5, NCBI_Transcript, Protein_name, Protein_sequence, length
2. `blastp_output_AF-ID_ncbi-proteins.filtered_best_hits.txt`: info on line 324 under header "BLASTp on NCBI Genome + AF IDs"
    - talk about headers beginning on line 614
3. `blastp_output_db_B_lac-SF5_q_blac-uniprot_on_WY-Domain_SP_CRN-motif_predicted-effectors-ov-85_RXLR-EER.tsv`: the table we've been dealing with for this entire experiment
    - headers: best_blast_hit_AF_ID, ORF_seq_ID, ...

I am going to concatenate files 1 and 3 by using the AF <-> NCBI information provided in file 2. First I need to extract just the AF to NCBI info from file 2:

```bash
awk -F'\t' '{print $1 "\t" $2}' blastp_output_AF-ID_ncbi-proteins.filtered_best_hits.txt | tail -n +2 > blac_WY_NCBI-seqs_linked.tsv
```

The resulting table has no headers, so I am going to use this Python shell script to give it the necessary headers:

```bash
python3
>>> import pandas as pd
>>> df = pd.read_table("blac_WY_NCBI-seqs_linked.tsv", sep='\t', header=0, names=["NCBI_ID_SF5", "best_blast_hit_AF_ID"]) # these are the headers
>>> df.to_csv("blac_WY_NCBI-seqs_linked_with-headers.tsv", sep='\t', index=False)
```

**TODO:** Update script so that it produces the list needed here (make new simple script that grabs this info)

<!--
I am going to use the Python shell with Pandas to link the two dataframes together

```bash
python3
>>> import pandas as pd
>>> df1 = pd.readcsv("Bremia-WY-NCBI-seqs.csv")
>>> df2 = pd.readcsv("blastp_output_db_B_lac-SF5_q_blac-uniprot_on_WY-Domain_SP_CRN-motif_predicted-effectors-ov-85_RXLR-EER.tsv", sep='\t')
>>> dflink = pd.readcsv("blac_WY_NCBI-seqs_linked.tsv", sep='\t')
>>> df_combine = 
```
-->

I created a script that links two tables together using a 2-column table as a linker. Here is the command using it:

```bash
python3 merge_tables_with_linker.py
```

### Unpickling a File

(12/6/2023)

Kelsey wanted me to unpickle a pickled file. I created the Python script `unpickle.py` that does this. I also ran this command which reformats the file to look better:

```bash
python3 unpickle.py features.pkl | less -S > features_unpickled.txt
```

### Getting AF Information from Kyle

I am trying to use the find command to find any databases that involves AlphaFold. Here is the command:

```bash
find /share/rwmwork/fletcher/ -exec grep -Ei '^(af|alphafold|afold)\w*$' {} + > fletcherdir_af_stuff.log
```

## Log

2/21/2024

- changed `create_ORF-ID_table.py` to `create_ORF_ID_table.py`
- changed `analyze_ORF-ID_table.py` to `analyze_ORF_ID_table.py`
