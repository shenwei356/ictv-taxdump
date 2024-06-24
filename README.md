# NCBI-style taxdump files for International Committee on Taxonomy of Viruses (ICTV)

Metagenomic tools like [Kraken2](https://github.com/DerrickWood/kraken2),
 [Centrifuge](https://github.com/DaehwanKimLab/centrifuge)
 and [KMCP](https://github.com/shenwei356/kmcp) support NCBI taxonomy in format of [NCBI taxdump files](https://ftp.ncbi.nlm.nih.gov/pub/taxonomy/).

While for virus, [ICTV](https://talk.ictvonline.org/) (International Committee on Taxonomy of Viruses) has its own taxonomy data.

A [TaxonKit](https://github.com/shenwei356/taxonkit) command, `taxonkit create-taxdump` is created
to create NCBI-style taxdump files for any taxonomy dataset,
including [GTDB](https://gtdb.ecogenomic.org/) and [ICTV](https://talk.ictvonline.org/).

Related projects:

- [gtdb-taxdump](https://github.com/shenwei356/ictv-taxdump): GTDB taxonomy taxdump files with trackable TaxIds
- [taxid-changelog](https://github.com/shenwei356/taxid-changelog): NCBI taxonomic identifier (taxid) changelog
- [taxonkit](https://github.com/shenwei356/taxonkit): A Practical and Efficient NCBI Taxonomy Toolkit

## Table of Contents

- [Table of Contents](#table-of-contents)
- [Download](#download)
- [Methods](#methods)
  - [Taxonomic hierarchy](#taxonomic-hierarchy)
  - [Generation of TaxIds](#generation-of-taxids)
  - [Data and tools](#data-and-tools)
  - [Steps](#steps)
- [Results](#results)
  - [Summary](#summary)
  - [Retrieving and reformating lineages](#retrieving-and-reformating-lineages)
  - [Taxa with only parts of ranks](#taxa-with-only-parts-of-ranks)
  - [More manipulations](#more-manipulations)
- [Citation](#citation)
- [Contributing](#contributing)
- [License](#license)


## Download

The [release page](https://github.com/shenwei356/ictv-taxdump/releases) contains taxdump files.

## Methods

### Taxonomic hierarchy

[Virus Metadata Resource (VMR)](https://talk.ictvonline.org/taxonomy/vmr/) provides taxonomy data of each release.
Most viruses have the seven-ranks (Kingdom, Phylum, Class, Order, Family, Genus, Species),
while some only have parts of the ranks, like Genus and Species.

### Generation of TaxIds

We hash the rank+taxon_name (in lower case) of each taxon node to `uint64`
using [xxhash](https://github.com/cespare/xxhash/) and convert it to `int32`.

See [more details](https://bioinf.shenwei.me/taxonkit/usage/#create-taxdump).

    
### Data and tools

The taxonomy data is released as a `.xlsx` file at https://talk.ictvonline.org/taxonomy/vmr/.

[TaxonKit](https://github.com/shenwei356/taxonkit) v0.12.0 or a later version is needed.
[v0.14.0](https://github.com/shenwei356/taxonkit/blob/master/CHANGELOG.md) or a later version is preferred.
**Since v0.14.0, [taxonkit create-taxdump](https://bioinf.shenwei.me/taxonkit/usage/#create-taxdump) stores
TaxIds in `int32` following BLAST and DIAMOND, rather than `uint32` in previous versions**.

[csvtk](https://github.com/shenwei356/csvtk) is used for data processing.

### Steps

    # download here: https://ictv.global/msl/current
    file="ICTV_Master_Species_List_2023_MSL39.v2.xlsx"
    sheet="MSL"

    # conver xlsx to tsv
    csvtk xlsx2csv "$file" -n $sheet \
        | csvtk csv2tab \
        > ictv.tsv
    
    # remove M-BM- characters.
    # https://askubuntu.com/questions/357248/how-to-remove-special-m-bm-character-with-sed
    sed -i 's/\xc2\xa0/ /g' ictv.tsv
    
    # not detected in MSL39
    #
    # remove leading and tailing blanks. e.g., "Escherichia phage PhaxI\t"
    # remove a newline character and a space introduced by accident
    csvtk replace -t -F -f "*" -p "^\s+|\s+$" ictv.tsv \
        | csvtk replace -t -F -f "*" -p "\n " -r "" \
        > ictv.clean.tsv
    
    # choose columns, rename, and remove duplicates
    csvtk cut -t ictv.clean.tsv -f "Realm,Subrealm,Kingdom,Subkingdom,Phylum,Subphylum,Class,Subclass,Order,Suborder,Family,Subfamily,Genus,Subgenus,Species" \
        | csvtk rename -t -f 1- -n "realm,subrealm,kingdom,subkingdom,phylum,subphylum,class,subclass,order,suborder,family,subfamily,genus,subgenus,species" \
        | csvtk uniq   -t -f 1- \
        > ictv.taxonomy.tsv
        
    # ------------------- create-taxdump -----------------------
    
    taxonkit create-taxdump ictv.taxonomy.tsv --out-dir ictv-taxdump/
    
    # set the environmental variable for taxonkit,
    # so we don't need specifiy "--data-dir ictv-taxdump" for each taxonkit command.
    export TAXONKIT_DB=ictv-taxdump


## Results

Set the environmental variable for taxonkit,
so we don't need specifiy `--data-dir ictv-taxdump` for each taxonkit command.

    export TAXONKIT_DB=ictv-taxdump

Check more [TaxonKit commands and usages](https://bioinf.shenwei.me/taxonkit/usage/)

### Summary

1. Count of all ranks (version: MSL39)
    
        $ taxonkit list --ids 1 \
            | taxonkit lineage -L -r \
            | csvtk freq -H -t -f 2 -n \
            | csvtk pretty -H -t

        no rank     1
        subphylum   2
        realm       6
        kingdom     10
        suborder    11
        phylum      18
        class       41
        order       81
        subgenus    84
        subfamily   200
        family      314
        genus       3522
        species     14690

    It perfectly matches the official data:

        Rank         MSL38 Total    New   Abolished   Moved   Renamed   MSL.39 Total
        ----------   -----------   ----   ---------   -----   -------   ------------
        Realm                  6      0           0       0         0              6
        Subrealm               0      0           0       0         0              0
        Kingdom               10      0           0       0         0             10
        Subkingdom             0      0           0       0         0              0
        Phylum                17      1           0       0         0             18
        Subphylum              2      0           0       0         0              2
        Class                 40      1           0       0         1             41
        Subclass               0      0           0       0         0              0
        Order                 72      9           0       0         3             81
        Suborder               8      3           0       0         0             11
        Family               264     51          -1      11         2            314
        Subfamily            182     18           0       1         1            200
        Genus               2818    820        -116      59         1           3522
        Subgenus              84      0           0       0         0             84
        Species            11273   3547        -130     407      2884          14690

        
### Retrieving and reformating lineages

1. The TaxId

        $ echo 'Betacoronavirus hongkongense' | taxonkit name2taxid
        Betacoronavirus hongkongense    418966335

1. Complete lineage

        $ echo 418966335 | taxonkit lineage
        418966335       Riboviria;Orthornavirae;Pisuviricota;Pisoniviricetes;Nidovirales;Cornidovirineae;Coronaviridae;Orthocoronavirinae;Betacoronavirus;Embecovirus;Betacoronavirus hongkongense

        # another format
        $ echo 418966335 \
            | taxonkit lineage -t \
            | csvtk cut -Ht -f 3 \
            | csvtk unfold -Ht -f 1 -s ";" \
            | taxonkit lineage -r -n -L \
            | csvtk cut -Ht -f 1,3,2 \
            | csvtk pretty -Ht

        1864891977   realm       Riboviria
        1844659726   kingdom     Orthornavirae
        38781089     phylum      Pisuviricota
        1832208221   class       Pisoniviricetes
        1393610206   order       Nidovirales
        218352182    suborder    Cornidovirineae
        779314330    family      Coronaviridae
        146452600    subfamily   Orthocoronavirinae
        68549826     genus       Betacoronavirus
        692402414    subgenus    Embecovirus
        418966335    species     Betacoronavirus hongkongense
        
        # in NCBI taxonomy
        $ echo 'Betacoronavirus' | taxonkit name2taxid --data-dir ~/.taxonkit \
            | csvtk cut -Ht -f 2 \
            | taxonkit lineage -t --data-dir ~/.taxonkit \
            | csvtk cut -Ht -f 3 \
            | csvtk unfold -Ht -f 1 -s ";" \
            | taxonkit lineage -r -n -L --data-dir ~/.taxonkit \
            | csvtk cut -Ht -f 1,3,2 \
            | csvtk pretty -Ht

        10239     superkingdom   Viruses
        2559587   clade          Riboviria
        2732396   kingdom        Orthornavirae
        2732408   phylum         Pisuviricota
        2732506   class          Pisoniviricetes
        76804     order          Nidovirales
        2499399   suborder       Cornidovirineae
        11118     family         Coronaviridae
        2501931   subfamily      Orthocoronavirinae
        694002    genus          Betacoronavirus
    
1. Reformat the lineage.

        # kingdom,phylum,class,order,family,genus,species
        $ echo 418966335 \
            | taxonkit reformat -I 1 -f "{K};{p};{c};{o};{f};{g};{s}"
        418966335       Orthornavirae;Pisuviricota;Pisoniviricetes;Nidovirales;Coronaviridae;Betacoronavirus;Betacoronavirus hongkongense

### Taxa with only parts of ranks

1. *Rhizidiomyces virus*.

        $ echo 'Rhizidiomyces virus' | taxonkit name2taxid
        Rhizidiomyces virus     1826348565

        $ echo 1826348565 | taxonkit lineage
        1826348565      Rhizidiovirus;Rhizidiomyces virus

        $ echo 1826348565 \
            | taxonkit lineage -t \
            | csvtk cut -Ht -f 3 \
            | csvtk unfold -Ht -f 1 -s ";" \
            | taxonkit lineage -r -n -L \
            | csvtk cut -Ht -f 1,3,2 \
            | csvtk pretty -Ht

        156661886    genus     Rhizidiovirus
        1826348565   species   Rhizidiomyces virus

1. *Deltasatellite solaniflavussecundi*.

        $ echo 'Deltasatellite solaniflavussecundi' | taxonkit name2taxid
        Deltasatellite solaniflavussecundi      574171172

        $ echo 574171172 | taxonkit lineage
        574171172       Tolecusatellitidae;Deltasatellite;Deltasatellite solaniflavussecundi

        $ echo 574171172 \
            | taxonkit lineage -t \
            | csvtk cut -Ht -f 3 \
            | csvtk unfold -Ht -f 1 -s ";" \
            | taxonkit lineage -r -n -L \
            | csvtk cut -Ht -f 1,3,2 \
            | csvtk pretty -Ht

        194525965   family    Tolecusatellitidae
        289873738   genus     Deltasatellite
        574171172   species   Deltasatellite solaniflavussecundi

### More manipulations

See https://bioinf.shenwei.me/taxonkit/usage/.

## Citation

> Shen, W., Ren, H., TaxonKit: a practical and efficient NCBI Taxonomy toolkit,
> Journal of Genetics and Genomics, [https://doi.org/10.1016/j.jgg.2021.03.006](https://www.sciencedirect.com/science/article/pii/S1673852721000837)

## Contributing

We welcome pull requests, bug fixes and issue reports.

## License

[MIT License](https://github.com/shenwei356/taxid-changelog/blob/master/LICENSE)
