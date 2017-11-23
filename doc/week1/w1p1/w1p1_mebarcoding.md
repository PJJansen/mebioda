Week 1 Data analysis of mycorrhizal diversity
---------------------------------------------

- **Step-by-step analysis of micorrhizal diversity**

1. Create a directory where you will store the files to use for the analyses, you can call it e.g. w1p1

2. **Cutadapt**

- Python package to trim the bad quality ends of the reads and the primers.

 [cutadapt.read.docs](https://cutadapt.readthedocs.io/en/v1.10/installation.html)     
   
- **Installation**     
        
        pip install --user --upgrade cutadapt

3. **Trimming the bad quality ends**

       mkdir out_trimmed

       for fq in *.fastq 
       do
	   cutadapt -q 20,20 -o out_trimmed/"${fq%.fastq}_trimmed_ends.fastq" ${fq}
       done


4. **Trimming the 3' primer**

        for fq in out_trimmed/*.fastq
        do 
        cutadapt -a TCCTCCGCTTATTGATAGC -o "${fq/_ends/_primers}" ${fq} 
        done 

5. **Trimming the 5' primer**
             
       for fq in out_trimmed/*_trimmed_primers.fastq
       do 
       cutadapt -g GTGARTCATCGAATCTTTG -o "${fq/_primers/_primers2}" ${fq}
       done


 Vsearch
 ------------
 Rognes et al., 2016. 
 
 [Vsearch github repository](https://github.com/torognes/vsearch)
 
 We have now the trimmed fastq file and we can proceed with the next steps:
 
1. **Quality filtering**

   
         for fq in out_trimmed/*_trimmed_primers2.fastq
         do
         vsearch --fastq_qmax 46 --fastq_filter ${fq} --fastq_maxee_rate 1 --fastq_trunclen 200 --fastaout "${fq%.fastq}.fa"
         done
         
- --fastq_qmax:specify the maximum quality score accepted when reading FASTQ files. 
- --fastq_maxee_rate 1: we specify the expected error per base: 1 
- --fastq_trunclen 200, all the sequences will be truncated at 200 bp; 

2. **Dereplicating at sample level and relabel with sample_n**

        cd out_trimmed
        for fq in *_trimmed_primers2.fa
        do
        vsearch --derep_fulllength  ${fq} --output "${fq/trimmed_primers2/uniques}" --relabel "${fq/trimmed_primers2/seq}" --sizeout 
        done 
        
- --derep_fulllength: Merge strictly identical sequences contained in filename.   
- --sizeout: the number of occurrences (i.e. abundance) of each sequence is indicated at the end of their fasta header.
- --relabel: sequence renaming is done to add sample prefixes to the sequence identifiers, so that we can later merge the sequences
        

3.  **Merging all the samples into one file**

        cat *uniques.fa* > CPuniques.fasta

4. **Dereplicating across samples and remove singletons**

       vsearch --derep_fulllength CPuniques.fasta --minuniquesize 2 --sizein --sizeout --fasta_width 0 --uc all.derep.uc --output CPuniq_no_sing.fasta

5. **Clustering at 97% before chimera detection**


       vsearch --cluster_size CPuniq_no_sing.fasta --id 0.97 --strand plus --sizein --sizeout --fasta_width 0 --uc CR_otus.uc --relabel OTU_ --centroids CP.otus.fasta --otutabout CP.otutab.txt
       
- --cluster_size: sorts sequences by decreasing abundance before
    clustering.
- --id: reject the sequence match if the pairwise identity is lower than 0.97
- --fasta_width: Fasta files produced by vsearch are wrapped (sequences are written on lines of integer
    nucleotides, 80 by default). To eliminate the wrapping, the value is set to zero.
- --centroids: The centroid is the sequence that seeded the cluster (i.e. the first sequence of the cluster).     
- --otutabout: Output an OTU table in the classic tab-separated plain text format as a matrix containing
the abundances of the OTUs in the different samples.
 

6. **De novo chimera detection**
            
       vsearch --uchime_denovo CR.otus.fasta --sizein --sizeout --fasta_width 0 --nonchimeras CR.otus.nonchim.fasta 

--uchime_denovo: Detect chimeras present in the fasta-formatted filename, without external references(i.e. de novo)       

7. **Comparing target sequences**

       vsearch --usearch_global CP.otus.nonchim.fasta -db utax_reference_dataset_10.10.2017.fasta -id 0.7 -blast6out CPotus.m8 -strand both -maxaccepts 1 -maxrejects 256
       
- --usearch_global: Compare target sequences (--db) to the fasta-formatted query sequences contained in
filename, using global pairwise alignment.

Part 2
-----------
We now have the all the output files that we need.

Open the CP.otutab.txt and the CPotus.m8 tables.

1. Open LibreOffice Calc and import the files separately, using the tab separated option.
 
2. In the CP.otus.m8 keep only the first two columns after the taxonomic information. They respectively contain the match percentage and the length of the sequences (bp). 

3. Sort the sheet according to the basepair column and delete all the sequences shorter than 150 bp. 

4. Then, insert a column after column A, click on the column A and go on Data -> Text to columns and use the semmi-comma separated option. You will see that the size information will be in the inserted column, you can remove this column.

5. Sort by column A and by the column containing the match percentages, using for this column the descending option. Move the last two columns of the sheet after column A.

6. Save both the files as text csv.

As you can see the CPotus.csv has duplicate rows which need to be removed.

The uniq.pl script provided will remove the duplicate rows 
 
7. Open the terminal:  

        perl uniq.pl CPotus.csv > CPotus.uniq.csv

We need now to join the two tables that we have created: 

             sort CP.tab.csv | more

             cat CP.tab.csv | sed -e 's/,/\t/'

             cat CP.tab.csv | sed -e 's/,/\t/' | sort | head

            cat CP.tab.csv | sed -e 's/,/\t/' | sort > CP.tab.sorted.csv

            sort CPotus_uniq.csv | head

            join CPotus_uniq.csv CP.tab.sorted.csv | more

            join CPotus_uniq.csv CP.tab.sorted.csv > CPotus.uniq.joined.csv
 
 
8. Open the file we just generated in LibreOffice Calc.

After the OTUs' abundance values calculate the sum. 
e.g =SUM(a1:g1)

Next to the sum we are going to write an IF condition that will return the value 1 if the abundance value is greater than 0, otherwise it will return 0. 
=IF(e1>0,1,0)

Finally, we want the sum of each absence-presence column, so at the end of each column calculate the sum.
