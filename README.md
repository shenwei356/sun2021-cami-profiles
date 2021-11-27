# Ground truth metagenomic profiles in CAMI format for Sun2021

> We
> simulated metagenomic sequencing reads for 25 communities
> from distinct habitats (for example, gastrointestinal, oral, dermal,
> vaginal and building, five communities for each habitat; Methods).
> To avoid reference database bias of different metagenomic profil-
> ers, the genomes used to generate simulated communities were
> selected from the intersection among the reference databases of
> MetaPhlAn2, mOTUs2 and Kraken2.
> 
> Sun, Z., Huang, S., Zhang, M. et al. Challenges in benchmarking metagenomic profilers. Nat Methods > 18, 618–626 (2021). https://doi.org/10.1038/s41592-021-01141-3

Sun *et al.* simulated 25 metagenomic reads for benchmarking metagenomic profilers,
while the ground truth profiles format is not convenient for interpretation
with tools like [opal](https://github.com/CAMI-challenge/OPAL). For example:

    $ head -n 4 sun/5_building_sequence_abd.txt 
    SpeciesID       sample_1        sample_2        sample_3        sample_4        sample_5
    Corynebacterium_jeikeium        0.00195744970771946     0       0       0.0081377431495817      0
    Lactococcus_lactis      0.0285317256732946      0       0.00218905863883039     0.00157454493494673     0.00769396428284639
    Streptococcus_agalactiae        0.00126070506577494     0       0.00268546083074535     0       0.00348204130934378

Taxonomic names instead of TaxId were used, and they were formated:

1. Square brackets were deleted:

        Orininal: [Clostridium] hiranonis
        Formated: Clostridium_hiranonis

2. Characters except letters and numbers were replaced with underlines.

        Orininal: Synechococcus sp. JA-2-3B'a(2-13)
        Formated: Synechococcus_sp_JA_2_3B_a_2_13

Besides, the verion of NCBI Taxonomy database was not clear. The only clue:

> Indeed, in the recently updated microbial genome database (NCBI RefSeq, 6 November 2020),

This made it hard to convert taxonomic names to the right TaxId,
because [NCBI Taxonomy changes frequently](https://github.com/shenwei356/taxid-changelog).
I had to manually checking the history of a taxon via taxid-changelog,
and finally found the lastest available taxdump version: `2020-06-01`.

## DOWNLOAD

- **Taxonomic abundance**: [sun2021_gs_taxonomic_abd.profile](https://github.com/shenwei356/sun2021-cami-profiles/blob/master/sun2021_gs_taxonomic_abd.profile)
- Sequence abundance: [sun2021_gs_sequence_abd.profile](https://github.com/shenwei356/sun2021-cami-profiles/blob/master/sun2021_gs_sequence_abd.profile)

## HOWTO

### Resourses

Datasets

- [Twenty-five simulated reads and ground truth profiles](https://figshare.com/projects/Pitfalls_and_Opportunities_in_Benchmarking_Metagenomic_Classifiers/79916).
- [taxdmp_2020-06-01](https://ftp.ncbi.nlm.nih.gov/pub/taxonomy/taxdump_archive/taxdmp_2020-06-01.zip)
- [taxdmp_2021-10-01](https://ftp.ncbi.nlm.nih.gov/pub/taxonomy/taxdump_archive/taxdmp_2021-10-01.zip) or other versions of NCBI taxdump files.

Tools:

- [tocami.py](https://github.com/hzi-bifo/cami2_pipelines/blob/master/bin/tocami.py), depending on Python package [ete3](http://etetoolkit.org/).
- [taxonkit](https://github.com/shenwei356/taxonkit)
- [rush](https://github.com/shenwei356/rush)
- [csvtk](https://github.com/shenwei356/csvtk)

### Preparing tocami.py

1. `tocami.py`:

        # wget https://raw.githubusercontent.com/hzi-bifo/cami2_pipelines/master/bin/tocami.py
        chmod a+x tocami.py
        
        # install pacakge ete3
        pip install ete3

2. Preparing `taxdump.tar.gz` for `tocami.py` (skip this if you download [taxdump.tar.gz](ftp://ftp.ncbi.nih.gov/pub/taxonomy/taxdump.tar.gz))

        # download taxdmp_2021-10-01.zip
        wget https://ftp.ncbi.nlm.nih.gov/pub/taxonomy/taxdump_archive/taxdmp_2021-10-01.zip
        
        # preparing taxdump.tar.gz for tocami.py
        unzip taxdmp_2021-10-01.zip names.dmp nodes.dmp merged.dmp delnodes.dmp 
        tar -zcvf taxdump.tar.gz names.dmp nodes.dmp merged.dmp delnodes.dmp
        
        # clean up
        /bin/rm *.dmp
        
3. Creating database for `ete3` (don't worry the error reports, just ignore):

        tocami.py -t taxdump.tar.gz -f motus -s 1 -d . sun/5_building_taxonomic_abd.txt 
        
        # output when creating taxa.sqlite
        Loading node names...
        2365884 names loaded.
        253670 synonyms loaded.
        Loading nodes...
        2365884 nodes loaded.
        Linking nodes...
        Tree is loaded.
        Updating database: ./taxa.sqlite ...
        2365000 generating entries... 
        Uploading to ./taxa.sqlite

        Inserting synonyms:      250000 
        Inserting taxid merges:  60000 
        Inserting taxids:       2365000

### Preparing mapping relationship between species names and TaxId

Sun mentioned:

> Indeed, in the recently updated microbial genome database (NCBI RefSeq, 6 November 2020),

After many attempts, I found the right version of the NCBI Taxonomy taxdump files, i.e., 2020-06-01.

    # wget https://ftp.ncbi.nlm.nih.gov/pub/taxonomy/taxdump_archive/taxdmp_2020-06-01.zip
    
    taxdump=taxdump
    
    mkdir -p $taxdump
    unzip taxdmp_2020-06-01.zip names.dmp nodes.dmp merged.dmp delnodes.dmp -d $taxdump
    
    # taxid -> name
    taxonkit list --ids 1 --indent "" --data-dir $taxdump \
        | taxonkit lineage -n -L --data-dir $taxdump \
        > taxid2name.tsv
    
    # Sun2021 deletes `[` and `]`
    # name -> taxid
    csvtk cut -lHt -f 2,1 taxid2name.tsv \
        | csvtk replace -lHt -f 1 -p '[\[\]]' \
        | csvtk replace -lHt -f 1 -p '[\W]+' -r '_' \
        > name2taxid.tsv
        

### Reformating

1. Changing `building` to `build` for name consistency. Because the reads file are:
`Build_sample1.left.fq.gz`,  `Gut_sample1.left.fq.gz`,  `Oral_sample1.left.fq.gz`,
`Skin_sample1.left.fq.gz`,  `VG_sample1.left.fq.gz`.


        # mv sun/5_building_sequence_abd.txt sun/5_build_sequence_abd.txt
        # mv sun/5_building_taxonomic_abd.txt sun/5_build_taxonomic_abd.txt

1. Reformating Sun's format to TIPP-like format:

        type=taxonomic_abd
        # type=sequence_abd
                
        rm -rf $type
        mkdir $type
        
        for f in sun/*$type.txt; do
            for c in $(seq 2 6); do 
                cut -f 1,$c $f \
                    | sed 1d \
                    | csvtk replace -Ht -K -k name2taxid.tsv -p '(.+)' -r '{kv}' \
                    > $type/$(basename $f | awk -F _ '{print $2}' | sed -r 's/^(.)/\U\1/')_sample$(expr $c - 1)           
            done 
        done
        
        
        # checking unsolved names:
        cat $type/* | csvtk grep -Ht -f 1 -p '[^\d]'
        
        # manually checking the change history via https://github.com/shenwei356/taxid-changelog
        
2. Formating to CAMI2 format:

        ls $type/* | rush './tocami.py -d ./ -f tipp {} -s {%} -o {}.profile'
        
3. Concatenating:

        cat $type/*.profile > sun2021_gs_$type.profile

    
