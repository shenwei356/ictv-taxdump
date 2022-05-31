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

* [Method](#method)
    + [Taxonomic hierarchy](#taxonomic-hierarchy)
    + [Generation of TaxIds](#generation-of-taxids)
    + [Data and tools](#data-and-tools)
    + [Steps](#steps)
* [Download](#download)
* [Results](#results)
    + [Summary](#summary)
    + [SARS-COV-2](#sars-cov-2)
    + [Taxa with same names](#taxa-with-same-names)
* [Citation](#citation)
* [Contributing](#contributing)
* [License](#license)

## Method

### Taxonomic hierarchy

[Virus Metadata Resource (VMR)](https://talk.ictvonline.org/taxonomy/vmr/) provides taxonomy data of each release.
Most viruses have the seven-ranks (Kingdom, Phylum, Class, Order, Family, Genus, Species), and 
each has a name. 

We assign the virus name a TaxId with the rank of "no rank" below the species rank.
Therefore, we can also track the changes of these assemblies via the TaxId later.

### Generation of TaxIds

We just hash the taxon name (in lower case) of each taxon node to `uint64`
using [xxhash](https://github.com/cespare/xxhash/) and convert it to `uint32`.

### Data and tools

The taxonomy data is released as a `.xlsx` file at https://talk.ictvonline.org/taxonomy/vmr/.

[TaxonKit](https://github.com/shenwei356/taxonkit) v0.12.0 or later version is needed.

### Steps
    
    # download here: https://talk.ictvonline.org/taxonomy/vmr/
    file="VMR_19-250422_MSL37.xlsx"
    
    # conver xlsx to tsv
    csvtk xlsx2csv "$file" \
        | csvtk csv2tab \
        > ictv.tsv
    
    # remove M-BM- characters.
    # https://askubuntu.com/questions/357248/how-to-remove-special-m-bm-character-with-sed
    sed -i 's/\xc2\xa0/ /g' ictv.tsv
    
    # remove leading and tailing blanks. e.g., "Escherichia phage PhaxI\t"
    # remove a newline character and a space introduced by accident
    csvtk replace -t -F -f "*" -p "^\s+|\s+$" ictv.tsv \
        | csvtk replace -t -F -f "*" -p "\n " -r "" \
        > ictv.clean.tsv
    
    # ------------------- create-taxdump -----------------------
    
    # Option A
    
    # choose columns, rename, and remove duplicates
    csvtk cut -t -f "Realm,Subrealm,Kingdom,Subkingdom,Phylum,Subphylum,Class,Subclass,Order,Suborder,Family,Subfamily,Genus,Subgenus,Species" ictv.clean.tsv \
        | csvtk rename -t -f 1-15 -n "realm,subrealm,kingdom,subkingdom,phylum,subphylum,class,subclass,order,suborder,family,subfamily,genus,subgenus,species" \
        | csvtk uniq   -t -f 1-15 \
        > ictv.taxonomy.tsv
        
    taxonkit create-taxdump -A 15 --field-accession-re "^(.+)$" \
        ictv.taxonomy.tsv --out-dir ictv-taxdump/

 
    # Option B: treating "Virus name(s)" as subpsecies
    
    # choose columns, rename, and remove duplicates
    csvtk cut -t -f "Realm,Subrealm,Kingdom,Subkingdom,Phylum,Subphylum,Class,Subclass,Order,Suborder,Family,Subfamily,Genus,Subgenus,Species,Virus name(s)" ictv.clean.tsv \
        | csvtk rename -t -f 1-16 -n "realm,subrealm,kingdom,subkingdom,phylum,subphylum,class,subclass,order,suborder,family,subfamily,genus,subgenus,species,subspecies" \
        | csvtk uniq   -t -f 1-16 \
        > ictv.taxonomy-with-subpsecies.tsv
        
    taxonkit create-taxdump -A 16 --field-accession-re "^(.+)$" \
        ictv.taxonomy-with-subpsecies.tsv --out-dir ictv-taxdump-with-subpsecies/


    
    # set the environmental variable for taxonkit,
    # so we don't need specifiy "--data-dir ictv-taxdump" for each taxonkit command.
    export TAXONKIT_DB=ictv-taxdump

## Download

The [release page](https://github.com/shenwei356/gtdb-taxdump/releases) contains taxdump files.

## Results

Set the environmental variable for taxonkit,
so we don't need specifiy "--data-dir ictv-taxdump" for each taxonkit command.

    export TAXONKIT_DB=ictv-taxdump-with-subpsecies

Check more [TaxonKit commands and usages](https://bioinf.shenwei.me/taxonkit/usage/)

### Summary

1. Count of all ranks (version: Virus Metadata Repository number 19, April 25 2022; MSL37)
    
        $ taxonkit list --ids 1 \
            | taxonkit lineage -L -r \
            | csvtk freq -H -t -f 2 -n \
            | csvtk pretty -H -t            
        no rank      1
        subphylum    2
        realm        6
        suborder     8
        kingdom      10
        phylum       17
        class        39
        order        65
        subgenus     84
        subfamily    168
        family       233
        genus        2606
        species      10434
        subspecies   10539
        
### SARS-COV-2

1. The TaxId

        $ grep 'severe acute respiratory syndrome coronavirus 2' ictv-taxdump-with-subpsecies/taxid.map 
        severe acute respiratory syndrome coronavirus 2        2363788870

1. Complete lineage

        $ echo 2363788870 | taxonkit lineage
        2363788870      Riboviria;Orthornavirae;Pisuviricota;Pisoniviricetes;Nidovirales;Cornidovirineae;Coronaviridae;Orthocoronavirinae;Betacoronavirus;Sarbecovirus;Severe acute respiratory syndrome-related coronavirus;severe acute respiratory syndrome coronavirus 2

        # another format
        $ echo 2363788870 \
            | taxonkit lineage -t \
            | csvtk cut -Ht -f 3 \
            | csvtk unfold -Ht -f 1 -s ";" \
            | taxonkit lineage -r -n -L \
            | csvtk cut -Ht -f 1,3,2 \
            | csvtk pretty -Ht
        492247681    realm        Riboviria
        104708768    kingdom      Orthornavirae
        1506901452   phylum       Pisuviricota
        3239177245   class        Pisoniviricetes
        37745009     order        Nidovirales
        2390145280   suborder     Cornidovirineae
        738421640    family       Coronaviridae
        1428506634   subfamily    Orthocoronavirinae
        906833049    genus        Betacoronavirus
        605136173    subgenus     Sarbecovirus
        1015862491   species      Severe acute respiratory syndrome-related coronavirus
        2363788870   subspecies   severe acute respiratory syndrome coronavirus 2
        
        # in NCBI taxonomy
        $ echo 2697049 \
            | taxonkit lineage -t --data-dir ~/.taxonkit \
            | csvtk cut -Ht -f 3 \
            | csvtk unfold -Ht -f 1 -s ";" \
            | taxonkit lineage -r -n -L \
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
        2509511   subgenus       Sarbecovirus
        694009    species        Severe acute respiratory syndrome-related coronavirus
        2697049   no rank        Severe acute respiratory syndrome coronavirus 2
    
### Taxa with same names

For examples species `Epseptimavirus ev329` and `Salmonella virus 329` both have a "Virus name" `Salmonella phage 3-29`.

        $ cat ictv.taxonomy-with-subpsecies.tsv \
            | csvtk grep -t -f subspecies -p 'Salmonella phage 3-29' \
            | csvtk cut -t -f genus,species,subspecies \
            | csvtk pretty -t
        genus            species                subspecies
        --------------   --------------------   ---------------------
        Epseptimavirus   Epseptimavirus ev329   Salmonella phage 3-29
        Epseptimavirus   Salmonella virus 329   Salmonella phage 3-29
        
They are assigned with different TaxIds and can be queried in `taxid.map`:

        $ grep 'Salmonella phage 3-29' ictv-taxdump-with-subpsecies/taxid.map 
        Salmonella phage 3-29   3010875164,3010875165
        
        $ grep 'Salmonella phage 3-29' ictv-taxdump-with-subpsecies/taxid.map \
            | csvtk unfold -Ht -f 2 -s , \
            | taxonkit lineage --data-dir ictv-taxdump-with-subpsecies/ -i 2 
        Salmonella phage 3-29   3010875164      Duplodnaviria;Heunggongvirae;Uroviricota;Caudoviricetes;Demerecviridae;Markadamsvirinae;Epseptimavirus;Epseptimavirus ev329;Salmonella phage 3-29
        Salmonella phage 3-29   3010875165      Duplodnaviria;Heunggongvirae;Uroviricota;Caudoviricetes;Demerecviridae;Markadamsvirinae;Epseptimavirus;Salmonella virus 329;Salmonella phage 3-29

## Citation

> Shen, W., Ren, H., TaxonKit: a practical and efficient NCBI Taxonomy toolkit,
> Journal of Genetics and Genomics, [https://doi.org/10.1016/j.jgg.2021.03.006](https://www.sciencedirect.com/science/article/pii/S1673852721000837)

## Contributing

We welcome pull requests, bug fixes and issue reports.

## License

[MIT License](https://github.com/shenwei356/taxid-changelog/blob/master/LICENSE)
