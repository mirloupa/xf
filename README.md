# *Xylella fastidiosa* (Xf) annotation and orthology search
Pipeline to create a phylogeny using 44 publicly available Xf genomes from GenBank. https://github.com/harrisonlab/Frankia used as guideline.
## 1. Download Xf genomic sequences from GenBank.
The FTP links to the currently 44 publicly available Xf genomes on GenBank (dated 11/2018) are saved in [xf-fna_genbank_links.txt](https://github.com/mirloupa/xf_phylogeny/blob/master/xf-fna_genbank_links.txt). Use the following bash script to download all sequences in the file to your genome directory:
```
for line in $(cat xf_genbank_links.txt); do
  wget $line /home/mirabl/Xf_proj/Ncbi_44/Xf_genomes
done
```
## 2. Unzip all downloaded sequences.
```
gunzip GCA_*
```
or
```
gzip -d GCA_*
```
## 3. Change FASTA file names to their GenBank accession numbers.
```
for file in /home/mirabl/Xf_proj/Ncbi_44/Xf_genomes/GCA_*; do
  mv /home/mirabl/Xf_proj/Ncbi_44/Xf_genomes/$file /home/mirabl/Xf_proj/Ncbi_44/Xf_genomes/$(head -1 $file | sed 's/ .*//' | sed 's/>//').fasta
done
```
## 4. Download annotation files (.gbff) from GenBank.
Repeat steps 1 and 2 for this, but instead using the [xf-gbff_genbank_links.txt](https://github.com/mirloupa/xf_phylogeny/blob/master/xf-gbff_genbank_links.txt) file, which contains FTP links to the annotation files.
## 5. Change the annotation file names to their GenBank accession numbers and replace the .gbff extension with .gbk.
```
for file in /home/mirabl/Xf_proj/Ncbi_44/Xf_genomes/*.gbff; do
  mv /home/mirabl/Xf_proj/Ncbi_44/Xf_genomes/$file /home/mirabl/Xf_proj/Ncbi_44/Xf_genomes/$(head -1 $file | tr -s ' ' | cut -d " " -f2).gbk
done
```
## 6. Create a Genus database using Prokka.
```
prokka-genbank_to_fasta_db /home/mirabl/Xf_proj/Ncbi_44/Xf_genomes/*.gbk > /home/mirabl/Xf_proj/Ncbi_44/Xf_genomes/xf_v2.faa
cd-hit -i /home/mirabl/Xf_proj/Ncbi_44/Xf_genomes/xf_v2.faa -o /home/mirabl/Xf_proj/Ncbi_44/Xf_genomes/xf_v2 -T 0 -M 0 -g 1 -s 0.8 -c 0.9
rm -fv /home/mirabl/Xf_proj/Ncbi_44/Xf_genomes/xf_v2.faa /home/mirabl/Xf_proj/Ncbi_44/Xf_genomes/xf_v2.bak.clstr /home/mirabl/Xf_proj/Ncbi_44/Xf_genomes/xf_v2.clstr
makeblastdb -dbtype prot -in /home/mirabl/Xf_proj/Ncbi_44/Xf_genomes/xf_v2
mkdir /home/mirabl/Xf_proj/Ncbi_44/DB_prokka
mv /home/mirabl/Xf_proj/Ncbi_44/Xf_genomes/xf_v2* /home/mirabl/Xf_proj/Ncbi_44/DB_prokka/
```
## 7. Run Prokka and compress (gzip) files.
#### See https://github.com/tseemann/prokka for documentation.
```
for file in /home/mirabl/Xf_proj/Ncbi_44/Xf_genomes/*.fasta; do
  file_short=$(basename $file | sed s/".fasta"//g)
  prokka --usegenus --genus /home/mirabl/Xf_proj/Ncbi_44/DB_prokka/xf_v2 $file --outdir /home/mirabl/Xf_proj/Ncbi_44/Xf_genomes/$file_short
  gzip /home/mirabl/Xf_proj/Ncbi_44/Xf_genomes/$file
done
```
#### Move all annotated directories to a new directory named 'Annotation':
```
mv /home/mirabl/Xf_proj/Ncbi_44/Xf_genomes/*.1 /home/mirabl/Xf_proj/Ncbi_44/Annotation/
mv /home/mirabl/Xf_proj/Ncbi_44/Xf_genomes/*.2 /home/mirabl/Xf_proj/Ncbi_44/Annotation/
```
## 8. Filter genomes based on Levy et al (2018) GWAS paper.
#### Run quast.py on all FASTA files
```
quast.py /home/mirabl/Xf_proj/Ncbi_44/Xf_genomes/*.fasta.gz
```
#### Filter genomes based on N50 >=40kbp and save only unique genomes into a new file:
```
python /home/hulinm/git_repos/tools/analysis/python_effector_scripts/extract_N50filtered_genomes.py /home/mirabl/Xf_proj/Ncbi_44/Xf_genomes/Quast_results/results_2018_12_10_14_08_42/transposed_report.tsv > /home/mirabl/Xf_proj/Ncbi_44/Xf_genomes/report2.txt
cut -f1 -d " " /home/mirabl/Xf_proj/Ncbi_44/Xf_genomes/report2.txt | uniq > /home/mirabl/Xf_proj/Ncbi_44/Xf_genomes/report3.txt 
```

#### Save reported genomes in a new directory named 'Filtered':
```
mkdir /home/mirabl/Xf_proj/Ncbi_44/Filtered
for file in /home/mirabl/Xf_proj/Ncbi_44/Xf_genomes/$(cat report3.txt); do
  cp /home/mirabl/Xf_proj/Ncbi_44/Xf_genomes/"$file".fasta /home/mirabl/Xf_proj/Ncbi_44/Filtered/
done
```
## 9. Run CheckM on filtered genomes from step 8.
#### CheckM can only be run on blacklace01 or blacklace 06. 
```
for file in /home/mirabl/Xf_proj/Ncbi_44/Filtered/*.fasta ; do
  file_short=$(basename $file | sed s/".fasta"//g) 
  echo $file_short 
  #mkdir -p /home/mirabl/Xf_proj/Ncbi_44/Filtered/Checkm/"$file_short"/Checkm 
  #cp $file /home/mirabl/Xf_proj/Ncbi_44/Filtered/Checkm/"$file_short" 
  Jobs=$(qstat | grep -i 'checkm' | wc -l) 
    while [ $Jobs -gt 5 ]; do 
      sleep 10
      printf "." 
      Jobs=$(qstat | grep -i 'checkm' | wc -l) 
    done
  qsub /home/mirabl/sub_checkm.pbs /home/mirabl/Xf_proj/Ncbi_44/Filtered/Checkm/"$file_short" /home/mirabl/Xf_proj/Ncbi_44/Filtered/Checkm/"$file_short"/Checkm 
done
```
#### Run CheckM report:
```
for file in /home/mirabl/Xf_proj/Ncbi_44/Filtered/*fasta; do
  file_short=$(basename $file | sed s/".fasta"//g)
  checkm qa /home/mirabl/Xf_proj/Ncbi_44/Filtered/Checkm/"$file_short"/Checkm/lineage.ms /home/mirabl/Xf_proj/Ncbi_44/Filtered/Checkm/"$file_short"/Checkm > Checkm/"$file_short"/Checkm/report
done
```
## 10. Perform orthology analysis on filtered, clean genomes using OrthoFinder.
Rename .faa files (from PROKKA output) to contain genome name not PROKKA output:
```
for file in /home/mirabl/Xf_proj/Ncbi_44/Xf_genomes/*.fasta.gz; do
  file_short=$(basename $file | sed s/".fasta.gz"//g)
  echo $file_short
  cp /home/mirabl/Xf_proj/Ncbi_44/Annotation/"$file_short"/*.faa /home/mirabl/Xf_proj/Ncbi_44/Annotation/"$file_short"/"$file_short".faa
done
```
## 11. Copy all .faa files to a new directory.
```
mkdir /home/mirabl/Xf_proj/Ncbi_44/OrthoFinder/Formatted
cp /home/mirabl/Xf_proj/Ncbi_44/Xf_genomes/*/*.faa /home/mirabl/Xf_proj/Ncbi_44/OrthoFinder/Formatted
```
## 12. Modify all fasta files to remove description, which is the correct format for OrthoMCL.
Each fasta item must be in format of strain|peg.number
```
for file in /home/mirabl/Xf_proj/Ncbi_44/OrthoFinder/Formatted/*.faa; do
  file_short=$(basename $file | sed s/".faa"//g | cut -f1 -d ".")
  echo $file_short
  sed -e 's/^(>[^[:space:]]).*/\1/' $file | sed s/"_"/"|peg."/g > /home/mirabl/Xf_proj/Ncbi_44/OrthoFinder/Formatted/"$file_short".fasta
done
```

```
for file in /home/hulinm/frankia/analysis/*.fasta; do
  id=$(less $file | grep ">" | cut -f1 -d "|" | sed s/">"//g | uniq) file_short=$(basename $file | sed s/".fasta"//g)
  echo $id
  echo $file_short
  sed s/"$id"/"$file_short"/g $file > $file_short.fasta
done
```
## 13. Remove manually those that did not pass CheckM and also those that did not pass N50 limit.
```
for file in /home/mirabl/Xf_proj/Ncbi_44/Filtered/*.fasta; do
  file_short=$(basename $file | sed s/".fasta"//g | cut -f1,2 -d _ | cut -f1 -d .)
  echo $file_short
  mv /home/hulinm/frankia/analysis/"$file_short".fasta /home/hulinm/frankia/analysis/orthofinder/formatted/"$file_short".fasta
done
```
## 14. Run OrthoFinder.
```
/home/hulinm/local/src/OrthoFinder-2.2.7_source/orthofinder/orthofinder.py -f formatted -t 16 -S diamond
```
## 15. Run BLAST on nod genes.
```
for GENOME in /home/hulinm/frankia/genomes/*.fna; do
  GENOME_FILE=$(basename $GENOME)
  GENOME_SHORT=$(echo $GENOME_FILE | sed s/.fna//g)
  echo $GENOME_SHORT
  python /home/hulinm/git_repos/tools/analysis/python_effector_scripts/rename.py -i "$GENOME_SHORT".fna -o "$GENOME_SHORT".fa
  gzip "$GENOME_SHORT".fna
  
  for genome in /home/hulinm/frankia/genomes/*.fa; do
    echo $genome
    file=$(basename $genome) genome_short=$(echo $file | sed s/.fa//g)
    echo $genome_short

    for query in /home/hulinm/frankia/nod_genes/*.fa; do
      echo $query
      query_short=$(basename $query | sed s/.fa//g)
  
    /home/hulinm/git_repos/tools/pathogen/blast/blast2csv.pl $query tblastn $genome 5 > /home/hulinm/frankia/nod_genes/blast/"$genome_short"_"$query_short"
    done
  done
done
```
