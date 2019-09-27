---
title: "Optimizing a Variant Calling Workflow"
teaching: 30
exercises: 15
questions:
- "How can I make my workflow more efficient and less error-prone?"
objectives:
- "Understand the different file systems "
- "Make sure we only keep need data"
keypoints:
- Bioinformatic tools are IO intensive, be aware of that.
---

# Compute Canada File Systems

### Some Bioinformatics truth

  - Many of the sequencing tools are IO intensive.
  - What _runs well_ on your laptop or one sample will not necessary scale to a 1000 subjects on a super computer.
  - There is a lot of data, a lot of it your will never be needed.

   To be able to run code with efficiency we need to have some information in mind about the HPC we run on.

### The file systems of Compute Canada Super Computer   

All the file system mounted on Béluga can bee seen by typing the `df -h` command. Here is a selection of what is available:


~~~
Filesystem                                     Size  Used Avail Use% Mounted on
rootfs                                          16G  2.0G   15G  13% /
tmpfs                                           47G  4.0G   43G   9% /tmp
pool/localscratch                              375G  256K  375G   1% /localscratch
10.72.112.11@o2ib:10.72.112.12@o2ib:/lustre02  105T  8.0T   97T   8% /lustre02 # /home folders
10.72.132.11@o2ib:10.72.132.12@o2ib:/lustre03  8.9P  6.8P  2.1P  77% /lustre03 # /project folder
10.72.112.13@o2ib:10.72.112.14@o2ib:/lustre04  2.6P  1.1P  1.6P  42% /lustre04 # /scratch folder
cvmfs2                                          44G   26G   19G  59% /cvmfs/soft.computecanada.ca # The software in "module"
cvmfs2                                          44G   26G   19G  59% /cvmfs/restricted.computecanada.ca # The licenced software  
cvmfs2                                          44G   26G   19G  59% /cvmfs/soft.mugqic Bioinformatic specific tools
~~~

https://docs.computecanada.ca/wiki/Storage_and_file_management

The luster file system have huge throughput, mush faster than any SSD. They archive that speed by having multiple copy of the data distributed over many physical hard drive. However, bioinfo tools are often opening and closing file at a fast rate and doing other operation that can put a lot of load on the luster file system. We often have whole cluster that are _jammed_ or rendered unusable for hours by bioinformatic ~~master students~~ researcher not following basic IO strategy.    


<object data="../img/good_practices.pdf" type="application/pdf" width="700px" height="700px">
    <embed src="../img/good_practices.pdf">
        <p>This browser does not support PDFs. Please download the PDF to view it: <a href="../img/good_practices.pdf">Download PDF</a>.</p>
    </embed>
</object>


These recommendations have short term and long term effects. I will mention the long term one, but we will implement here one that will prevent you from being that student that crash one of the biggest super computer in the world.

### Long term effects: use `/scratch`


  The project space size is in between 1 to 10 TB. It should be use to store input (raw) data and to store your results at the end of a project. This space is backed up on all Compute Canada systems.

  Use the scratch space. The scratch space is there to store temporary files and has a large default quota (20TB, can be p to 100TB). You can store there all the intermediate file that you need to run you code but that you do not have to keep in the long run. A cleaning is occurring for files older than 60 days. Once you are happy about what append in scratch keep a record of the test that where executed to get your data, move what is needed to your analysis to the project space and delete everything else.

  In the long term, that will help you not reaching your quota limit on the project space, you will have documented what you are doing, will be sure that what is left is important to some degree and you will not have your PHD deleted by error!

### Short term effects: use `/localscratch`

Every Béluga node has a NVMe ssh mounted at `/localscratch`. This file system is not shared like `/scratch` and `/project`, is only accessible by its compute node. It lets you spread the io load of yours jobs. There is nothing there when you start your code, it will be emptied when you code is done running.

The typical procedure is:
* Move the input data to the `/localscratch`.
* Use it as the folder for your output (working directory).
* Move the data back to the shared file system.



### Run a Workflows that uses `/localscratch`


In this exercise we will develop a complete variant calling workflow that will make good use of the `/localscratch`. But first how should we access it? Once a job starts to run on a compute node a directory is created in `/localscratch`. The full path of that directory in stored in the $SLURM_TMPDIR/. This variable does not exist on the login node, this can be sometime annoying, but Compute Canada might implement something to fix that.

In an interactive session, we do have access to the variable:

~~~
$ salloc --time=0-2:0  --cpus-per-task=1 --mem=2300  --account=def-training-wa
salloc: Pending job allocation 2978942
salloc: job 2978942 queued and waiting for resources
salloc: job 2978942 has been allocated resources
salloc: Granted job allocation 2978942
salloc: Waiting for resource configuration
salloc: Nodes blg4810 are ready for job
[poq@blg4810 ~]$]$ echo $SLURM_TMPDIR/
/localscratch/poq.2978942.0/
~~~
{: .bash}

The path is `/localscratch/${USENAME}.${JOBID}/`.





<!--# Analyzing Quality with FastQC

We will use the command `touch` to create a new file where we will write our shell script. We will create this script in a new
directory called `scripts/`. Previously, we used
`nano` to create and open a new file. The command `touch` allows us to create a new file without opening that file.

~~~
$ mkdir -p ~/dc_workshop/scripts
$ cd ~/dc_workshop/scripts
$ touch read_qc.sh
$ ls
~~~
{: .bash}

~~~
read_qc.sh
~~~
{: .output}

We now have an empty file called `read_qc.sh` in our `scripts/` directory. We will now open this file in `nano` and start
building our script.

~~~
$ nano read_qc.sh
~~~
{: .bash}

**Enter the following pieces of code into your shell script**

Our first line will ensure that our script will exit if an error occurs, and is sometimes a good idea to include at the beginning of your scripts. The second line will move us into the `untrimmed_fastq/` directory when we run our script.

~~~
set -e
cd ~/dc_workshop/data/untrimmed_fastq/
~~~
{: .output}

These next two lines will give us a status message to tell us that we are currently running FastQC, then will run FastQC
on all of the files in our current directory with a `.fastq` extension.

~~~
echo "Running FastQC ..."
fastqc *.fastq*
~~~
{: .output}

Our next line will create a new directory to hold our FastQC output files. Here we are using the `-p` option for `mkdir`. This
option allows `mkdir` to create the new directory, even if one of the parent directories doesn't already exist. It also supresses
errors if the directory already exists, without overwriting that directory. It is a good idea to use this option in your shell
scripts to avoid running into errors if you don't have the directory structure you think you do.

~~~
mkdir -p ~/dc_workshop/results/fastqc_untrimmed_reads
~~~
{: .output}

Our next three lines first give us a status message to tell us we are saving the results from FastQC, then moves all of the files
with a `.zip` or a `.html` extension to the directory we just created for storing our FastQC results.

~~~
echo "Saving FastQC results..."
mv *.zip ~/dc_workshop/results/fastqc_untrimmed_reads/
mv *.html ~/dc_workshop/results/fastqc_untrimmed_reads/
~~~
{: .output}

The next line moves us to the results directory where we've stored our output.

~~~
cd ~/dc_workshop/results/fastqc_untrimmed_reads/
~~~
{: .output}

The next five lines should look very familiar. First we give ourselves a status message to tell us that we're unzipping our ZIP
files. Then we run our for loop to unzip all of the `.zip` files in this directory.

~~~
echo "Unzipping..."
for filename in *.zip
    do
    unzip $filename
    done
~~~
{: .output}

Next we concatenate all of our summary files into a single output file, with a status message to remind ourselves that this is
what we're doing.

~~~
echo "Saving summary..."
cat */summary.txt > ~/dc_workshop/docs/fastqc_summaries.txt
~~~
{: .output}

> ## Using `echo` statements
>
> We've used `echo` statements to add progress statements to our script. Our script will print these statements
> as it is running and therefore we will be able to see how far our script has progressed.
>
{: .callout}

Your full shell script should now look like this:

~~~
set -e
cd ~/dc_workshop/data/untrimmed_fastq/

echo "Running FastQC ..."
fastqc *.fastq*

mkdir -p ~/dc_workshop/results/fastqc_untrimmed_reads

echo "Saving FastQC results..."
mv *.zip ~/dc_workshop/results/fastqc_untrimmed_reads/
mv *.html ~/dc_workshop/results/fastqc_untrimmed_reads/

cd ~/dc_workshop/results/fastqc_untrimmed_reads/

echo "Unzipping..."
for filename in *.zip
    do
    unzip $filename
    done

echo "Saving summary..."
cat */summary.txt > ~/dc_workshop/docs/fastqc_summaries.txt
~~~
{: .output}

Save your file and exit `nano`. We can now run our script:

~~~
$ bash read_qc.sh
~~~
{: .bash}

~~~
Running FastQC ...
Started analysis of SRR2584866.fastq
Approx 5% complete for SRR2584866.fastq
Approx 10% complete for SRR2584866.fastq
Approx 15% complete for SRR2584866.fastq
Approx 20% complete for SRR2584866.fastq
Approx 25% complete for SRR2584866.fastq
.
.
.
~~~
{: .output}

For each of your sample files, FastQC will ask if you want to replace the existing version with a new version. This is
because we have already run FastQC on this samples files and generated all of the outputs. We are now doing this again using
our scripts. Go ahead and select `A` each time this message appears. It will appear once per sample file (six times total).

~~~
replace SRR2584866_fastqc/Icons/fastqc_icon.png? [y]es, [n]o, [A]ll, [N]one, [r]ename:
~~~
{: .output}
-->

# Automating our Variant Calling Workflow

We have already wrote a script to run `fastqc` and trimmomatic in a previous chapters. We will now do so now up the the variant calling output. This will be all done while using the `/localscratch` file system.




~~~
#!/bin/bash
#SBATCH --cpu-per-task 4
#SBATCH --mem-per-cpu  4775
#SBATCH -A def-poq-tr


if [ $# -ne 3 ]; then
   echo "$0 <input dir> <output_dir> <input fastqs BASENAME>"
   exit  1
fi

INPUT_DIR=$1
OUTPUT_DIR=$2
BASENAME=$3

set -e

# Know where you are at the beginning of the run
START_DIR=$PWD

INPUT_PAIR1=$INPUT_DIR/${BASENAME}_1.fastq.gz
INPUT_PAIR2=$INPUT_DIR/${BASENAME}_2.fastq.gz

# With "set -e" activated this will wake the code exit if one of the pair is not present
ls "$INPUT_PAIR1" "$INPUT_PAIR2"

echo $INPUT_PAIR1 and $INPUT_PAIR2 exist
echo Starting process


# Get the input in the tmp dir and move there too
# Sometime you do not need to move the inputs to the tmpdir.
mkdir -p $SLURM_TMPDIR/$INPUT_DIR
cp $INPUT_PAIR1 $INPUT_PAIR2 $SLURM_TMPDIR/$INPUT_DIR
cd  $SLURM_TMPDIR

TRIM_OUT=trimm_out

mkdir -p ${TRIM_OUT}

TRIM1=${TRIM_OUT}/${BASENAME}_1.trim.fastq.gz
UNTRIM1=${TRIM_OUT}/${BASENAME}_1un.trim.fastq.gz

TRIM2=${TRIM_OUT}/${BASENAME}_2.trim.fastq.gz
UNTRIM2=${TRIM_OUT}/${BASENAME}_2un.trim.fastq.gz


module load trimmomatic

if [ ! -f $START_DIR/$OUTPUT_DIR/$TRIM1 ] ; then
  echo $START_DIR/$OUTPUT_DIR/$TRIM1
  java -jar $EBROOTTRIMMOMATIC/trimmomatic-0.36.jar \
   PE -threads $SLURM_CPUS_PER_TASK  $INPUT_PAIR1 $INPUT_PAIR2 \
   $TRIM1 $UNTRIM1 \
   $TRIM2 $UNTRIM2 \
   SLIDINGWINDOW:4:20 MINLEN:25 \
   ILLUMINACLIP:$EBROOTTRIMMOMATIC/adapters/NexteraPE-PE.fa:2:40:15
else
  echo not recomputing trim for $BASENAME
  cp $START_DIR/$OUTPUT_DIR/{$TRIM1,$TRIM2,$UNTRIM2,$UNTRIM1} ${TRIM_OUT}

fi



genome=/cvmfs/ref.mugqic/genomes/species/Escherichia_coli_str_k_12_substr_dh10b.ASM1942v1/genome/bwa_index/Escherichia_coli_str_k_12_substr_dh10b.ASM1942v1.fa


# The outputs paths
sam=sam/${BASENAME}.aligned.sam
bam=bam/${BASENAME}.aligned.bam
sorted_bam=bam/${BASENAME}.aligned.sorted.bam
raw_bcf=bcf/${BASENAME}_raw.bcf
variants=bcf/${BASENAME}_variants.vcf
final_variants=vcf/${BASENAME}_final_variants.vcf

mkdir -p vcf bcf bam sam

module load bwa
bwa mem -t $SLURM_CPUS_PER_TASK  $genome $TRIM1 $TRIM2 > $sam
module load samtools
samtools view --threads $SLURM_CPUS_PER_TASK  -S -b $sam > $bam
samtools sort --threads $SLURM_CPUS_PER_TASK -o $sorted_bam $bam
samtools index -@ $SLURM_CPUS_PER_TASK $sorted_bam


module load bcftools
genome=/cvmfs/ref.mugqic/genomes/species/Escherichia_coli_str_k_12_substr_dh10b.ASM1942v1/genome/Escherichia_coli_str_k_12_substr_dh10b.ASM1942v1.fa
bcftools mpileup -O b -o $raw_bcf -f $genome $sorted_bam
bcftools call --threads $SLURM_CPUS_PER_TASK --ploidy 1 -m -v -o $variants $raw_bcf
vcfutils.pl varFilter $variants > $final_variants


# We only keep the trimmed fastq and bam. We can always recompute the
# the samcfiles.
mkdir -p  $START_DIR/$OUTPUT_DIR

echo syching to $START_DIR/$OUTPUT_DIR
rsync -rltv vcf bcf trimm_out bam $START_DIR/$OUTPUT_DIR


~~~
{: .bash}

<!--
Our variant calling workflow has the following steps:

1. Index the reference genome for use by bwa and samtools.
2. Align reads to reference genome.
3. Convert the format of the alignment to sorted BAM, with some intermediate steps.
4. Calculate the read coverage of positions in the genome.
5. Detect the single nucleotide polymorphisms (SNPs).
6. Filter and report the SNP variants in VCF (variant calling format).

Let's go through this script together:

~~~
$ cd ~/dc_workshop/scripts
$ less run_variant_calling.sh
~~~
{: .bash}

The script should look like this:

~~~
set -e
cd ~/dc_workshop/results

genome=~/dc_workshop/data/ref_genome/ecoli_rel606.fasta

bwa index $genome

mkdir -p sam bam bcf vcf

for fq1 in ~/dc_workshop/data/trimmed_fastq_small/*_1.trim.sub.fastq
    do
    echo "working with file $fq1"

    base=$(basename $fq1 _1.trim.sub.fastq)
    echo "base name is $base"

    fq1=~/dc_workshop/data/trimmed_fastq_small/${base}_1.trim.sub.fastq
    fq2=~/dc_workshop/data/trimmed_fastq_small/${base}_2.trim.sub.fastq
    sam=~/dc_workshop/results/sam/${base}.aligned.sam
    bam=~/dc_workshop/results/bam/${base}.aligned.bam
    sorted_bam=~/dc_workshop/results/bam/${base}.aligned.sorted.bam
    raw_bcf=~/dc_workshop/results/bcf/${base}_raw.bcf
    variants=~/dc_workshop/results/bcf/${base}_variants.vcf
    final_variants=~/dc_workshop/results/vcf/${base}_final_variants.vcf

    bwa mem $genome $fq1 $fq2 > $sam
    samtools view -S -b $sam > $bam
    samtools sort -o $sorted_bam $bam
    samtools index $sorted_bam
    bcftools mpileup -O b -o $raw_bcf -f $genome $sorted_bam
    bcftools call --ploidy 1 -m -v -o $variants $raw_bcf
    vcfutils.pl varFilter $variants > $final_variants

    done
~~~
{: .output}

Now, we'll go through each line in the script before running it.

First, notice that we change our working directory so that we can create new results subdirectories
in the right location.

~~~
cd ~/dc_workshop/results
~~~
{: .output}

Next we tell our script where to find the reference genome by assigning the `genome` variable to
the path to our reference genome:

~~~
genome=~/dc_workshop/data/ref_genome/ecoli_rel606.fasta
~~~
{: .output}

Next we index our reference genome for BWA:

~~~
bwa index $genome
~~~
{: .output}

And create the directory structure to store our results in:

~~~
mkdir -p sam bam bcf vcf
~~~
{: .output}

Then, we use a loop to run the variant calling workflow on each of our FASTQ files. The full list of commands
within the loop will be executed once for each of the FASTQ files in the `data/trimmed_fastq/` directory.
We will include a few `echo` statements to give us status updates on our progress.

The first thing we do is assign the name of the FASTQ file we're currently working with to a variable called `fq1` and
tell the script to `echo` the filename back to us so we can check which file we're on.

~~~
for fq1 in ~/dc_workshop/data/trimmed_fastq_small/*_1.trim.sub.fastq
    do
    echo "working with file $fq1"
~~~
{: .bash}

We then extract the base name of the file (excluding the path and `.fastq` extension) and assign it
to a new variable called `base`.
~~~
    base=$(basename $fq1 _1.trim.sub.fastq)
    echo "base name is $base"
~~~
{: .bash}

We can use the `base` variable to access both the `base_1.fastq` and `base_2.fastq` input files, and create variables to store the names of our output files. This makes the script easier to read because we don't need to type out the full name of each of the files: instead, we use the `base` variable, but add a different extension (e.g. `.sam`, `.bam`) for each file produced by our workflow.


~~~
    #input fastq files
    fq1=~/dc_workshop/data/trimmed_fastq_small/${base}_1.trim.sub.fastq
    fq2=~/dc_workshop/data/trimmed_fastq_small/${base}_2.trim.sub.fastq

    # output files
    sam=~/dc_workshop/results/sam/${base}.aligned.sam
    bam=~/dc_workshop/results/bam/${base}.aligned.bam
    sorted_bam=~/dc_workshop/results/bam/${base}.aligned.sorted.bam
    raw_bcf=~/dc_workshop/results/bcf/${base}_raw.bcf
    variants=~/dc_workshop/results/bcf/${base}_variants.vcf
    final_variants=~/dc_workshop/results/vcf/${base}_final_variants.vcf     
~~~
{: .bash}


And finally, the actual workflow steps:

1) align the reads to the reference genome and output a `.sam` file:

~~~
    bwa mem $genome $fq1 $fq2 > $sam
~~~
{: .output}

2) convert the SAM file to BAM format:

~~~
    samtools view -S -b $sam > $bam
~~~
{: .output}

3) sort the BAM file:

~~~
    samtools sort -o $sorted_bam $bam
~~~
{: .output}

4) index the BAM file for display purposes:

~~~
    samtools index $sorted_bam
~~~
{: .output}

5) calculate the read coverage of positions in the genome:

~~~
    bcftools mpileup -O b -o $raw_bcf -f $genome $sorted_bam
~~~
{: .output}

6) call SNPs with bcftools:

~~~
    bcftools call --ploidy 1 -m -v -o $variants $raw_bcf
~~~
{: .output}

7) filter and report the SNP variants in variant calling format (VCF):

~~~
    vcfutils.pl varFilter $variants  > $final_variants
~~~
{: .output}



> ## Exercise
> It's a good idea to add comments to your code so that you (or a collaborator) can make sense of what you did later.
> Look through your existing script. Discuss with a neighbor where you should add comments. Add comments (anything following
> a `#` character will be interpreted as a comment, bash will not try to run these comments as code).
{: .challenge}


Now we can run our script:

~~~
$ bash run_variant_calling.sh
~~~
{: .bash}


> ## Exercise
>
> The samples we just performed variant calling on are part of the long-term evolution experiment introduced at the
> beginning of our variant calling workflow. From the metadata table, we know that SRR2589044 was from generation 5000,
> SRR2584863 was from generation 15000, and SRR2584866 was from generation 50000. How did the number of mutations per sample change
> over time? Examine the metadata table. What is one reason the number of mutations may have changed the way they did?
>
> Hint: You can find a copy of the output files for the subsampled trimmed FASTQ file variant calling in the
> `~/.solutions/wrangling-solutions/variant_calling_auto/` directory.
>
>> ## Solution
>>
>> ~~~
>> $ for infile in ~/dc_workshop/results/vcf/*_final_variants.vcf
>> > do
>> >     echo ${infile}
>> >     grep -v "#" ${infile} | wc -l
>> > done
>> ~~~
>> {: .bash}
>>
>> For SRR2589044 from generation 5000 there were 10 mutations, for SRR2584863 from generation 15000 there were 25 mutations,
>> and SRR2584866 from generation 766 mutations. In the last generation, a hypermutable phenotype had evolved, causing this
>> strain to have more mutations.
> {: .solution}
{: .challenge}


> ## Bonus Exercise
>
> If you have time after completing the previous exercise, use `run_variant_calling.sh` to run the variant calling pipeline
> on the full-sized trimmed FASTQ files. You should have a copy of these already in `~/dc_workshop/data/trimmed_fastq`, but if
> you don't, there is a copy in `~/.solutions/wrangling-solutions/trimmed_fastq`. Does the number of variants change per sample?
{: .challenge}
-->
