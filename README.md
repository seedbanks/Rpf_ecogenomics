## using edirect to get fastas according to organism names
1. get organism names from IMG taxa infor for the genome cart.

2. run edirect:   
```
cd /mnt/data3/fan/rpf/April2016_ana

while read line; do echo "/mnt/data3/fan/edirect/esearch -db assembly -query '"'"'$line'"[Organism]'"'| /mnt/data3/fan/edirect/elink -target nuccore -Name assembly_nuccore_refseq | /mnt/data3/fan/edirect/efetch -db nuccore -format fasta > ${line// /_}.txt"; done < img_actino_org_names.txt > edirect_org_name_get_fa_commands.sh

cd /mnt/data3/fan/rpf/April2016_ana/img_actinos/without_filter_fas

cat ../../edirect_org_name_get_fa_commands.sh | parallel
```

3. remove empty files  
```
find . -size 0 -delete
```





1. get html files from img for ncbi number scraping
```
while read oid; do echo 'curl --cookie cookies -A "User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10.9; rv:32.0) Gecko/20100101 Firefox/32.0" "https://img.jgi.doe.gov/cgi-bin/m/main.cgi?section=TaxonDetail&page=taxonDetail&taxon_oid='$oid'" > oid_html/img.'$oid'.out'; done < img_oids.txt > ../get_oid_html.txt
```

```
cat ../get_oid_html.txt | parallel
```

2. parsing out the ncbi numbers
```
cd /mnt/data5/fan/rpf/April2016_ana/oid_html
grep "<a href='http://www.ncbi.nlm.nih.gov/entrez/viewer.fcgi?" *.out > oid_to_ncbi.txt
```

3. in R, find out the oid's that are present in the jgi portal downloads. Write to file "oid_to_ignore_from_ncbi_list.txt". Using vi to replace "\n" with " -e " and create file "oid_to_ignore_from_ncbi_list2.txt".   

4. negative grep to remove oid's that should be ignored:   
```
while read line; do grep -v $line oid_to_ncbi.txt; done < oid_to_ignore_from_ncbi_list2.txt > oid_to_ncbi_to_ignore_removed.txt
```

5. generate single ncbi accession number for each oid:  
```
python /mnt/data3/fan/code/parsers/img_oid_html_ncbi_parser.py oid_to_ncbi_to_ignore_removed.txt > final_oid_to_ncbi_acc_number.txt
```

6. using edirect to pull out genbank files from ncbi acc nuber, and delete empty files: 
```
cut -f 2 final_oid_to_ncbi_acc_number.txt > ncbi_acc.txt

while read line; do echo "/mnt/data3/fan/edirect/efetch -db nuccore -id $line -format gb > $line.gbk"; done < ncbi_acc.txt > edirect_ncbi_acc_get_gbk_commands.sh

cd genbank_files
cat  ../edirect_ncbi_acc_get_gbk_commands.sh | parallel

find . -size 0 -delete
```

7. shotgun metagenome contigs are not stored under the same accession number. To pull the assembled contigs, we need to get the information from the genbank file. In genbank file folder, separate metagenomes and annotated genbank files. Pull sequence contigs for metagenomes for the associated accession number.   
```
mkdir metagenomes
mv *000000.gbk metagenomes/

mkdir full_genomes
mv *.gbk full_genomes/

cd metagenomes
grep -w ^WGS *.gbk > ../metagenome_scafolds.txt
```

8. check the number of lines. There is one organism have multiple "WGS" lines. Find it and remove using vi.  
```
 cut -d : -f 1 metagenome_scafolds.txt | uniq -c | grep -w 2
```

9. created folders for individual organisms metagenome acc number. get individual contig acc for each metagenome.
```
cd /mnt/data3/fan/rpf/April2016_ana/oid_html/fasta_files/metagenomes

python /mnt/data3/fan/code/seq_util/ncbi_wgs_scaffold_to_fasta.py ../../genbank_files/metagenome_scafolds.txt

bash /mnt/data3/fan/code/seq_util/edirect_for_diff_folders.sh
cat ../edirect_contig_acc_get_fa_commands.sh | parallel

bash /mnt/data3/fan/code/seq_util/hmmsearch_command.sh
```

10. directly get rna sequences from assemblies from ncbi.   
```
cd /mnt/scratch/yangfan1/rpf/April2016_ana/oid_html/rna_fas
## from refseq assembly
cat ../ncbi_acc.txt | epost -db nuccore -format acc | elink -target assembly | efetch -format docsum | xtract -pattern DocumentSummary -element FtpPath_RefSeq | awk -F"/" '{print $0"/"$NF"_rna_from_genomic.fna.gz"}' | xargs wget
## from genbank assembly
cat ../ncbi_acc.txt | epost -db nuccore -format acc | elink -target assembly | efetch -format docsum | xtract -pattern DocumentSummary -element FtpPath_GenBank | awk -F"/" '{print $0"/"$NF"_rna_from_genomic.fna.gz"}' | xargs wget
```

11. or direclty get feature table with gene positions in assemblies.   
```
cd /mnt/scratch/yangfan1/rpf/April2016_ana/oid_html/rna_fas

cat ../ncbi_acc.txt | epost -db nuccore -format acc | elink -target assembly | efetch -format docsum | xtract -pattern DocumentSummary -element FtpPath_RefSeq | awk -F"/" '{print $0"/"$NF"_feature_table.txt.gz"}' | xargs wget

cat ../ncbi_acc.txt | epost -db nuccore -format acc | elink -target assembly | efetch -format docsum | xtract -pattern DocumentSummary -element FtpPath_GenBank | awk -F"/" '{print $0"/"$NF"_feature_table.txt.gz"}' | xargs wget

## linking assembly database with ncbi acc number
while read line; do echo $line; echo $line | epost -db nuccore -format acc | elink -target assembly | efetch -format docsum | xtract -pattern DocumentSummary -element FtpPath_GenBank; done < ../ncbi_acc.txt >> output.txt

## wget save using acc number as file name:
while read line; do echo $line | epost -db nuccore -format acc | elink -target assembly | efetch -format docsum | xtract -pattern DocumentSummary -element FtpPath_GenBank |awk -F"/" '{print $0"/"$NF"_rna_from_genomic.fna.gz"}' | xargs wget -O $line.rna.fa.gz; done < ../ncbi_acc.txt

``` 

13. using bioproject ids:
```
cd by_bioprojectid

while read line; do echo $line | epost -db bioproject | elink -target assembly | efetch -format docsum | xtract -pattern DocumentSummary -element FtpPath_GenBank |awk -F"/" '{print $0"/"$NF"_rna_from_genomic.fna.gz"}' | xargs wget -O $line.rna.fa.gz; done < ncbi_bioproject_id.txt 
```

### if can't find rna in genbank assembly, retry in refseq assembly:
while read line; do echo $line | epost -db bioproject | elink -target assembly | efetch -format docsum | xtract -pattern DocumentSummary -element FtpPath_RefSeq |awk -F"/" '{print $0"/"$NF"_rna_from_genomic.fna.gz"}' | xargs wget -O $line.rna.fa.gz; done < retry.txt

### for the rest of the oid's that don't have a ncbi related id, jgi portal can't access. download use organism names. 
```
python ~/repos/code/parsers/edirect_org_name_name_oid_command.py ../../last_oid_strain_names.txt > edirect_org_name_oid.sh 

cat edirect_org_name_oid.sh | parallel
```

14. parsing extracted bundles from img_portal
```
cd /mnt/home/yangfan1/rpf/April2016_ana/img_actinos/by_img_oid/from_jgi_portal/extracted
ls -d * > list_genome_oid.txt
while read line; do python ~/repos/code/parsers/img_gff_16S_to_fna.py $line/$line.gff $line/$line.genes.fna ../no_16s/$line.error > ../all_16S/$line.16S.fna; done < list_genome_oid.txt
```

15. pull the longest 16S sequences from jgi portal gff extracted 16S files:
```
cd /mnt/home/yangfan1/rpf/April2016_ana/img_actinos/by_img_oid/from_jgi_portal/all_16S
python ~/repos/code/seq_util/pull_longest_seq_from_img_gff_matched_fa.py 500 jgi_portal_all_16S_no_16S.error *.16S.fna > jgi_portal_all_16S_longest_min500.fa
```

16. pull the longest 16S sequences using keywords search from ncbi.rna.fa.gz files (downloaded using edirect). for example:
```
cd /mnt/home/yangfan1/rpf/April2016_ana/img_actinos/by_org_name/refseq_assem

python ~/repos/code/seq_util/pull_16s_from_ncbi.rna.fa.gz_files.py "16S|SSU|ssu" 500 ../refseq_assem_no_16s.error *.rna.fa.gz > ../refseq_assem_longest_16S_min500.fa
```

17. pull the longest 16S with min lenth of 500 out of the everything that was direclty downloaded from img website:
```
cd /mnt/home/yangfan1/rpf/April2016_ana/last_oid_downloaded_directly_from_img

python ~/repos/code/seq_util/pull_longest_seq_from_img_fa.py last_oid_combined_16s_genecart.txt last_oid_combined_16s_na.fa 500 last_oid_combined_no_16S_min500.error > last_oid_combined_longest_16S_min500.fa
```

18. alignment using infernal and rdp bacterial model
```
/mnt/research/rdp/public/thirdParty/infernal-1.1/src/cmalign -o min500.stk -g --noprob /mnt/scratch/yangfan1/databases/RDPinfernal1.1Traindata/bacteria_model.cm < combined_longest_16S_min500.fa 
```

19. convert stk to fa:
```
mkdir temp
mv min500.stk temp
java -Xmx10g -jar /mnt/research/rdp/public/RDPTools/AlignmentTools.jar alignment-merger temp min500.fa
```
