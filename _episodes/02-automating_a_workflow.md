---
title: "Automating a Variant Calling Workflow"
teaching: 0
exercises: 0
questions:
- "How can I automate this?"
objectives:
- "Understand what a shell script is"
- "Automate an analytical workflow"
-  "Group a series of sequential commands into a script to automate a workflow"
keypoints:
---
# What is a shell script?
A shell script is basically a text file that contains a list of commands
that are executed sequentially. The commands in a shell script are the same as you would use on the command line.

Once you have worked out the details and tested your commands in the shell, you can save them into a file so, the next time, you
can automate the process with a script.

The basic anatomy of a shell script is a file with a list of commands.
That is also the definition of pretty much any computer program.


    #!/bin/bash

    cd ~/dc_sample_data

    for file in untrimmed_fastq/*.fastq
    do
        echo "My file name is $file"
    done


This looks a lot like the for loops we saw earlier. In fact, it is no different, apart from using indentation and the lack of the
'>' prompts; it's just saved in a text file. The line at the top ('#!/bin/bash') is commonly called the shebang line, which is a
special kind of comment that tells the shell which program is to be used as the 'intepreter' that executes the code.  

In this case, the interpreter is bash, which is the shell environment we are working in. The same approach is also used for other
scripting languages such as perl and python. 

# How to run a shell script
There are two ways to run a shell script the first way is to specify the
interpreter (bash) and the name of the script.  By convention, shell script
use the .sh extension, though this is not enforced.


    $ bash myscript.sh
    My file name is untrimmed_fastq/SRR097977.fastq
    My file name is untrimmed_fastq/SRR098026.fastq

The second was is a little more complicated to set up.

The first step, which only needs to be done once, is to modify the file 'permissions' of the text file so that the shell knows
the file is executable.

    $ chmod +x myscript.sh

After that, you can run the script as a regular program by just typing it's name on the command line.

    $ ./myscript.sh
    My file name is untrimmed_fastq/SRR097977.fastq
    My file name is untrimmed_fastq/SRR098026.fastq

The thing about running programs on the command line is that the shell may not know the location of your executables unless they 
are in the 'path' of know locations for programs.  So, you need to tell the shell the path to your script, which is './' if it is 
in the same directory.


> ## Exercise
> 1) Use nano to save the code above to a script called myscript.sh  
> 2) run the script  
{: .challenge}


# A real shell script

Now, let's do something real.  First, recall the code from our our fastqc
workflow from this morning, with a few extra "echo" statements.

    cd ~/dc_workshop/data/untrimmed_fastq/

    echo "Running fastqc..."
    ~/FastQC/fastqc *.fastq
    mkdir -p ~/dc_workshop/results/fastqc_untrimmed_reads

    echo "saving..."
    mv *.zip ~/dc_workshop/results/fastqc_untrimmed_reads/
    mv *.html ~/dc_workshop/results/fastqc_untrimmed_reads/

    cd ~/dc_workshop/results/fastqc_untrimmed_reads/

    echo "Unzipping..."
    for zip in *.zip
        do
        unzip $zip
        done

    echo "saving..."
    cat */summary.txt > ~/dc_workshop/docs/fastqc_summaries.txt

> ## Exercise
> 1) Use nano to create a shell script using with the code above (you can copy/paste), named read_qc.sh  
> 2) Run the script  
> 3) Bonus points: Use something you learned yesterday to save the output of the script to a file while it is running.  
{: .challenge}


To get started with this lesson, we will need to grab some data from an outside
server using `wget` on the command line.

Make sure you are in the `dc_workshop` directory first

    $ cd ~/dc_sample_data
    $ ls 
    $ r_genomics  sra_metadata  untrimmed_fastq  variant_calling.tar.gz

The file 'variant_calling.tar.gz' is what is commonly called a "tarball", which is
a compressed archive similar to the .zip files we have seen before.  We can decompress
this archive using the command below.


    $ tar -zxvf variant_calling.tar.gz

This will create a directory tree that contains some input data (reference genome and fastq files)
and a shell script that details the series of commands used to run the variant calling workflow.

<pre>
$ tree variant_calling

variant_calling
├── data
│   ├── ref_genome
│   │   └── ecoli_rel606.fasta
│   └── trimmed_fastq
│       ├── SRR097977.fastq
│       ├── SRR098026.fastq
│       ├── SRR098027.fastq
│       ├── SRR098028.fastq
│       ├── SRR098281.fastq
│       └── SRR098283.fastq
└── run_variant_calling.sh

3 directories, 8 files

</pre>

Without getting into the details yet, the variant calling workflow will do the following steps

1. Index the reference genome for use by bwa and samtools
2. Align reads to reference genome
3. Convert the format of the alignment to sorted BAM, with some intermediate steps.
4. Calculate the read coverage of positions in the genome
5. Detect the single nucleotide polymorphisms (SNPs)
6. Filter and report the SNP variants in VCF (variant calling format)

Let's walk through the commands in the workflow

The first command is to change to our working directory
so the script can find all the files it expects

    $ cd ~/dc_workshop/variant_calling

Assign the name/location of our reference genome
to a variable ($genome)

    $ genome=data/ref_genome/ecoli_rel606.fasta

>**Tip:** Within the Bash shell you can create variables at any time (as we did above, and during the 'for' loop lesson). Assign and name and the value using the assignment operator: '='. You can check the the shell knows the definition of your variable by typing: echo $variable_name. 

We need to index the reference genome for bwa and samtools. bwa
and samtools are programs that are pre-installed on our server.

    bwa index $genome
    samtools faidx $genome

Create output paths for various intermediate and result files The -p option means mkdir will create the whole path if it does not exist (no error or message will give given if it does exist)

    $ mkdir -p results/sai
    $ mkdir -p results/sam
    $ mkdir -p results/bam
    $ mkdir -p results/bcf
    $ mkdir -p results/vcf

We will now use a loop to run the variant calling work flow of each of our fastq files, so the list of command below will be execute once for each fastq files.

We would start the loop like this, so the name of each fastq file will by assigned to $fq

    $ for fq in data/trimmed_fastq/*.fastq
    > do
    > # etc...


In the script, it is a good idea to use echo for debugging/reporting to the screen

    $ echo "working with file $fq"

This command will extract the base name of the file
(without the path and .fastq extension) and assign it
to the $base variable

    $ base=$(basename $fq .fastq)
    $ echo "base name is $base"

We will assign various file names to variables both
for convenience but also to make it easier to see what 
is going on in the commands below.

    $ fq=data/trimmed_fastq/$base\.fastq
    $ sai=results/sai/$base\_aligned.sai
    $ sam=results/sam/$base\_aligned.sam
    $ bam=results/bam/$base\_aligned.bam
    $ sorted_bam=results/bam/$base\_aligned_sorted.bam
    $ raw_bcf=results/bcf/$base\_raw.bcf
    $ variants=results/bcf/$base\_variants.bcf
    $ final_variants=results/vcf/$base\_final_variants.vcf    

Our data are now staged.  The series of command below will run the steps of the analytical workflow

Align the reads to the reference genome

    $ bwa aln $genome $fq > $sai

Convert the output to the SAM format

    $ bwa samse $genome $sai $fq > $sam

Convert the SAM file to BAM format

    $ samtools view -S -b $sam > $bam

Sort the BAM file

    $ samtools sort -f $bam $sorted_bam

Index the BAM file for display purposes

    $ samtools index $sorted_bam

Do the first pass on variant calling by counting
read coverage

    $ samtools mpileup -g -f $genome $sorted_bam > $raw_bcf

Do the SNP calling with bcftools

    $ bcftools view -bvcg $raw_bcf > $variants

Filter the SNPs for the final output

    $ bcftools view $variants | /usr/share/samtools/vcfutils.pl varFilter - > $final_variants

> ## Exercise
> Run the script: dcuser/dc_sample_data/variant_calling/run_variant_calling.sh
> 
> $ bash run_variant_calling.sh
{: .challenge}






