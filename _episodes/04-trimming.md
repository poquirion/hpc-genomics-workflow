---
title: "Trimming and Filtering"
teaching: 30
exercises: 15
questions:
- "How can I get rid of sequence data that doesn't meet my quality standards?"
objectives:
- "Clean FASTQ reads using Trimmomatic."
- "Select and set multiple options for command-line bioinformatic tools."
- "Write `for` loops with two variables."
keypoints:
- "The options you set for the command-line tools you use are important"
- "You need to check that the scheduler are in phase with them."
---

# Cleaning Reads

In the previous episode, we took a high-level look at the quality
of each of our samples using FastQC. We visualized per-base quality
graphs showing the distribution of read quality at each base across
all reads in a sample and extracted information about which samples
fail which quality checks. Some of our samples failed quite a few quality metrics used by FastQC. This doesn't mean,
though, that our samples should be thrown out! It's very common to have some quality metrics fail, and this may or may not be a problem for your downstream application. For our variant callig workflow, we will be removing some of the low quality sequences to reduce our false positive rate due to sequencing error.

We will use a program called
[Trimmomatic](http://www.usadellab.org/cms/?page=trimmomatic) to
filter poor quality reads and trim poor quality bases from our samples.

## Trimmomatic Options

Trimmomatic has a variety of options to trim your reads. Note that this is a java code. It can be tricky to run, but the module command give us gidence.
Running the command will give us even more information.

~~~
$ module load trimmomatic
To execute Trimmomatic run: java -jar $EBROOTTRIMMOMATIC/trimmomatic-0.36.jar
$ java -jar $EBROOTTRIMMOMATIC/trimmomatic-0.36.jar
~~~
{: .bash}

Gives you the following output:
~~~
Usage:
       PE [-version] [-threads <threads>] [-phred33|-phred64] [-trimlog <trimLogFile>] [-summary <statsSummaryFile>] [-quiet] [-validatePairs] [-basein <inputBase> | <inputFile1> <inputFile2>] [-baseout <outputBase> | <outputFile1P> <outputFile1U> <outputFile2P> <outputFile2U>] <trimmer1>...
   or:
       SE [-version] [-threads <threads>] [-phred33|-phred64] [-trimlog <trimLogFile>] [-summary <statsSummaryFile>] [-quiet] <inputFile> <outputFile> <trimmer1>...
   or:
       -version
~~~
{: .output}

This output shows us that we must first specify whether we have paired end (`PE`) or single end (`SE`) reads.
Next, we specify what flag we would like to run. For example, you can specify `threads` to indicate the number of
processors on your computer that you want Trimmomatic to use. These flags are not necessary, but they can give
you more control over the command. The flags are followed by positional arguments, meaning the order in which you
specify them is important. In paired end mode, Trimmomatic expects the two input files, and then the names of the
output files. These files are described below.

| option    | meaning |
| ------- | ---------- |
|  \<inputFile1>  | Input reads to be trimmed. Typically the file name will contain an `_1` or `_R1` in the name.|
| \<inputFile2> | Input reads to be trimmed. Typically the file name will contain an `_2` or `_R2` in the name.|
|  \<outputFile1P> | Output file that contains surviving pairs from the `_1` file. |
|  \<outputFile1U> | Output file that contains orphaned reads from the `_1` file. |
|  \<outputFile2P> | Output file that contains surviving pairs from the `_2` file.|
|  \<outputFile2U> | Output file that contains orphaned reads from the `_2` file.|

The last thing trimmomatic expects to see is the trimming parameters:

| step   | meaning |
| ------- | ---------- |
| `ILLUMINACLIP` | Perform adapter removal. |
| `SLIDINGWINDOW` | Perform sliding window trimming, cutting once the average quality within the window falls below a threshold. |
| `LEADING`  | Cut bases off the start of a read, if below a threshold quality.  |
|  `TRAILING` |  Cut bases off the end of a read, if below a threshold quality. |
| `CROP`  |  Cut the read to a specified length. |
|  `HEADCROP` |  Cut the specified number of bases from the start of the read. |
| `MINLEN`  |  Drop an entire read if it is below a specified length. |
|  `TOPHRED33` | Convert quality scores to Phred-33.  |
|  `TOPHRED64` |  Convert quality scores to Phred-64. |

We will use only a few of these options and trimming steps in our
analysis. It is important to understand the steps you are using to
clean your data. For more information about the Trimmomatic arguments
and options, see [the Trimmomatic manual](http://www.usadellab.org/cms/uploads/supplementary/Trimmomatic/TrimmomaticManual_V0.32.pdf).

However, a complete command for Trimmomatic will look something like the command below. This command is an example and will not work, as we do not have the files it refers to:

~~~
#!/bin/bash
java -jar $EBROOTTRIMMOMATIC/trimmomatic-0.36.jar \
              PE -threads 4 SRR_1056_1.fastq SRR_1056_2.fastq  \
              SRR_1056_1.trimmed.fastq SRR_1056_1un.trimmed.fastq \
              SRR_1056_2.trimmed.fastq SRR_1056_2un.trimmed.fastq \
              ILLUMINACLIP:SRR_adapters.fa SLIDINGWINDOW:4:20
~~~
{: .bash}

In this example, we've told Trimmomatic:

| code   | meaning |
| ------- | ---------- |
| `PE` | that it will be taking a paired end file as input |
| `-threads 4` | to use four computing threads to run (this could speed up our run, if we know what we are doing!) |
| `SRR_1056_1.fastq` | the first input file name |
| `SRR_1056_2.fastq` | the second input file name |
| `SRR_1056_1.trimmed.fastq` | the output file for surviving pairs from the `_1` file |
| `SRR_1056_1un.trimmed.fastq` | the output file for orphaned reads from the `_1` file |
| `SRR_1056_2.trimmed.fastq` | the output file for surviving pairs from the `_2` file |
| `SRR_1056_2un.trimmed.fastq` | the output file for orphaned reads from the `_2` file |
| `ILLUMINACLIP:SRR_adapters.fa`| to clip the Illumina adapters from the input file using the adapter sequences listed in `SRR_adapters.fa` |
|`SLIDINGWINDOW:4:20` | to use a sliding window of size 4 that will remove bases if their phred score is below 20 |

## Running Trimmomatic

Now we will run Trimmomatic on our data. To begin, navigate to your `untrimmed_fastq` data directory:

~~~
$ cd ~/dc_workshop/data/untrimmed_fastq
~~~
{: .bash}

We are going to run Trimmomatic on one of our paired-end samples.
While using FastQC we saw that Nextera adapters were present in our samples.
The adapter sequences came with the installation of trimmomatic, so we will first copy these sequences into our current directory.

~~~
$ ls $EBROOTTRIMMOMATIC/adapters/
NexteraPE-PE.fa  TruSeq2-PE.fa  TruSeq2-SE.fa  TruSeq3-PE-2.fa  TruSeq3-PE.fa  TruSeq3-SE.fa
~~~
{: .bash}

We will also use a sliding window of size 4 that will remove bases if their
phred score is below 20 (like in our example above). We will also
discard any reads that do not have at least 25 bases remaining after
this trimming step the here is the `trimm.sh` script.

~~~
#!/bin/bash

INPUT_PAIR1=untrimmed_fastq/SRR2589044_1.fastq.gz
INPUT_PAIR2=untrimmed_fastq/SRR2589044_2.fastq.gz
OUTPUT_DIR=trimm_out

mkdir -p trimm_out

sample=${OUTPUT_DIR}/$(basename $INPUT_PAIR1)
TRIM1=${sample%%.fastq.gz}.trim.fastq.gz
UNTRIM1=${sample%%.fastq.gz}un.trim.fastq.gz

sample=${OUTPUT_DIR}/$(basename $INPUT_PAIR2)
TRIM2=${sample%%.fastq.gz}.trim.fastq.gz
UNTRIM2=${sample%%.fastq.gz}un.trim.fastq.gz


java -jar $EBROOTTRIMMOMATIC/trimmomatic-0.36.jar \
      PE $INPUT_PAIR1 $INPUT_PAIR2 \
      $TRIM1 $UNTRIM1 \
      $TRIM2 $UNTRIM2 \
      SLIDINGWINDOW:4:20 MINLEN:25 \
      ILLUMINACLIP:$EBROOTTRIMMOMATIC/adapters/NexteraPE-PE.fa:2:40:15


~~~
{: .bash}


>## Exercise
>
>Before submitting the script with `sbatch`, look at `sbatch` documentation and try to find a way to accelerate the processing.
>Think about the commend that was made about threads.  
>
>Hint: Look for the `cpu` keyword in the documentation
>
>>## Solution
>>You will need to ad threads to the code and to the sbatch commands
>>~~~
>>#!/bin/bash
>>
>>
>>INPUT_PAIR1=untrimmed_fastq/SRR2589044_1.fastq.gz
>>INPUT_PAIR2=untrimmed_fastq/SRR2589044_2.fastq.gz
>>OUTPUT_DIR=trimm_out
>>
>>mkdir -p trimm_out
>>
>>sample=${OUTPUT_DIR}/$(basename $INPUT_PAIR1)
>>TRIM1=${sample%%.fastq.gz}.trim.fastq.gz
>>UNTRIM1=${sample%%.fastq.gz}un.trim.fastq.gz
>>
>>sample=${OUTPUT_DIR}/$(basename $INPUT_PAIR2)
>>TRIM2=${sample%%.fastq.gz}.trim.fastq.gz
>>UNTRIM2=${sample%%.fastq.gz}un.trim.fastq.gz
>>
>>
>>java -jar $EBROOTTRIMMOMATIC/trimmomatic-0.36.jar
>>      PE -threads 4  $INPUT_PAIR1 $INPUT_PAIR2 \
>>      $TRIM1 $UNTRIM1 \
>>      $TRIM2 $UNTRIM2 \
>>      SLIDINGWINDOW:4:20 MINLEN:25 \
>>      ILLUMINACLIP:$EBROOTTRIMMOMATIC/adapters/NexteraPE-PE.fa:2:40:15
>>
>>
>>~~~
>>{: .bash}
>>Submit with the '--cpu-per-task' option:
>>
>>~~~
>>$ sbatch --cpu-per-task 4 trimm.sh
>>~~~
>>{: .bash}
>>In the slurm.<JOBID>.out you will find the following
>>~~~
>>TrimmomaticPE: Started with arguments:
>> SRR2589044_1.fastq.gz SRR2589044_2.fastq.gz SRR2589044_1.trim.fastq.gz SRR2589044_1un.trim.fastq.gz SRR2589044_2.trim.fastq.gz
>>SRR2589044_2un.trim.fastq.gz SLIDINGWINDOW:4:20 MINLEN:25 ILLUMINACLIP:NexteraPE-PE.fa:2:40:15
>>Multiple cores found: Using 2 threads
>>Using PrefixPair: 'AGATGTGTATAAGAGACAG' and 'AGATGTGTATAAGAGACAG'
>>Using Long Clipping Sequence: 'GTCTCGTGGGCTCGGAGATGTGTATAAGAGACAG'
>>Using Long Clipping Sequence: 'TCGTCGGCAGCGTCAGATGTGTATAAGAGACAG'
>>Using Long Clipping Sequence: 'CTGTCTCTTATACACATCTCCGAGCCCACGAGAC'
>>Using Long Clipping Sequence: 'CTGTCTCTTATACACATCTGACGCTGCCGACGA'
>>ILLUMINACLIP: Using 1 prefix pairs, 4 forward/reverse sequences, 0 forward only sequences, 0 reverse only sequences
>>Quality encoding detected as phred33
>>Input Read Pairs: 1107090 Both Surviving: 885220 (79.96%) Forward Only Surviving: 216472 (19.55%) Reverse Only Surviving: 2850 (0.26%) Dropped: 2548 (0.23%)
>>TrimmomaticPE: Completed successfully
>>~~~
>>{: .output}
> {: .solution}
{: .challenge}

### Slurm Digression

There is a second method to feed options to an `sbatch` script. The options can be embedded directly inside the script. Here is how we would write the   `trimm.sh` to follow that method:

~~~
#!/bin/bash
#SBATCH --cpu-per-task 4

INPUT_PAIR1=untrimmed_fastq/SRR2589044_1.fastq.gz
INPUT_PAIR2=untrimmed_fastq/SRR2589044_2.fastq.gz
OUTPUT_DIR=trimm_out

mkdir -p trimm_out

sample=${OUTPUT_DIR}/$(basename $INPUT_PAIR1)
TRIM1=${sample%%.fastq.gz}.trim.fastq.gz
UNTRIM1=${sample%%.fastq.gz}un.trim.fastq.gz

sample=${OUTPUT_DIR}/$(basename $INPUT_PAIR2)
TRIM2=${sample%%.fastq.gz}.trim.fastq.gz
UNTRIM2=${sample%%.fastq.gz}un.trim.fastq.gz


java -jar $EBROOTTRIMMOMATIC/trimmomatic-0.36.jar
      PE -threads 4  $INPUT_PAIR1 $INPUT_PAIR2 \
      $TRIM1 $UNTRIM1 \
      $TRIM2 $UNTRIM2 \
      SLIDINGWINDOW:4:20 MINLEN:25 \
      ILLUMINACLIP:$EBROOTTRIMMOMATIC/adapters/NexteraPE-PE.fa:2:40:15


~~~
{: .bash}
Submit with the '--cpu-per-task' option:

Then no need to explicitely add the option in the sbatch call.
~~~
$ sbatch  trimm.sh
~~~




One cpu (here a core) is the default that will be given to a script sent by sbatch. There are other default values. Along with the number of cpu/cores, the other important value is the RAM reserved for a job.  


We has a hint of these default values when we where looking the the outputs of `squeue -u $USER`.


~~~
          JOBID     USER      ACCOUNT           NAME  ST  TIME_LEFT NODES CPUS       GRES MIN_MEM NODELIST (REASON)
        2931031      poq def-poq-ab_c        trim.sh  PD    1:00:00     1    1     (null)    256M  (Priority)
~~~
{: .output}


They give a 1h walltime, the job will be killed after 1 h (--time 1:00:00), you get one Node, for most (95%) genomics jobs you will only be able to run on one node at a time (-N 1), we already mentioned that you get one cpu (and one task for that matter, so --cpu-per-task 1) and the memory is pretty small!
(--mem-per-cpu 256M).



>## Exercise
> Béluga has three type of node, 95000MB, 191000MB and 771000MB. What would be your --mem-per-cpu entry to get a proportional share of these nodes. Give your solution in MB or in GB.
>>## Solution
>>2375 MB  = 2.34 GB  
>>4775 MB = 4.66 GB  
>>19275  MB = 18.82GB  
>> If you gave your solution in GB, note that 1024 MB == 1 GB. I would recommend that you always use MB when you request memory on CC systems.
> {: .solution}
{: .challenge}


Note that once the job are ran on a compute node all the job submit info is available under SLURM_* environment variable. For example, the amount of memory and the number of core are stored under SLURM_MEM_PER_NODE, and SLURM_CPUS_PER_TASK respectively.


>## Exercises
>Rewrite the trimm.sh script so the number of cores does not have to be written twice. Add the right amount of memory so the job will run on the 191000MB nodes.
>>## Solution
>>
>>~~~
>>#!/bin/bash
>>#SBATCH --cpu-per-task 4
>>#SBATCH --mem-per-cpu  4775
>>
>>INPUT_PAIR1=untrimmed_fastq/SRR2589044_1.fastq.gz
>>INPUT_PAIR2=untrimmed_fastq/SRR2589044_2.fastq.gz
>>OUTPUT_DIR=trimm_out
>>
>>mkdir -p trimm_out
>>
>>sample=${OUTPUT_DIR}/$(basename $INPUT_PAIR1)
>>TRIM1=${sample%%.fastq.gz}.trim.fastq.gz
>>UNTRIM1=${sample%%.fastq.gz}un.trim.fastq.gz
>>
>>sample=${OUTPUT_DIR}/$(basename $INPUT_PAIR2)
>>TRIM2=${sample%%.fastq.gz}.trim.fastq.gz
>>UNTRIM2=${sample%%.fastq.gz}un.trim.fastq.gz
>>
>>
>>java -jar $EBROOTTRIMMOMATIC/trimmomatic-0.36.jar
>>      PE -threads $SLURM_CPUS_PER_TASK  $INPUT_PAIR1 $INPUT_PAIR2 \
>>      $TRIM1 $UNTRIM1 \
>>      $TRIM2 $UNTRIM2 \
>>      SLIDINGWINDOW:4:20 MINLEN:25 \
>>      ILLUMINACLIP:$EBROOTTRIMMOMATIC/adapters/NexteraPE-PE.fa:2:40:15
>>
>>
>>~~~
>>{: .bash}
>{: .solution}
{: .challenge}



### Back to Trimmomatic output

We can confirm that we have our output files. The output files are also FASTQ files. It should be smaller than our
input file, because we've removed reads. We can confirm this:


~~~
$ ls -hl  */SRR2589044*.fastq.gz
~~~
{: .bash}

~~~
-rw-rw-r-- 1 poq def-poq-ab  94M Sep 19 11:24 trimm_out/SRR2589044_1.trim.fastq.gz
-rw-rw-r-- 1 poq def-poq-ab  18M Sep 19 11:24 trimm_out/SRR2589044_1un.trim.fastq.gz
-rw-rw-r-- 1 poq def-poq-ab  91M Sep 19 11:24 trimm_out/SRR2589044_2.trim.fastq.gz
-rw-rw-r-- 1 poq def-poq-ab 271K Sep 19 11:24 trimm_out/SRR2589044_2un.trim.fastq.gz
-rw-rw-r-- 1 poq def-poq-ab 124M Sep  9 14:58 untrimmed_fastq/SRR2589044_1.fastq.gz
-rw-rw-r-- 1 poq def-poq-ab 128M Sep  9 14:59 untrimmed_fastq/SRR2589044_2.fastq.gz
~~~
{: .output}



We've just successfully run Trimmomatic on one of our FASTQ files!
However, there is some bad news. Trimmomatic can only operate on
one sample at a time and we have more than one sample.



<!--The good news
is that we can use a `for` loop to iterate through our sample files
quickly!

We unzipped one of our files before to work with it, let's compress it again before we run our for loop.

~~~
gzip SRR2584863_1.fastq
~~~
{: .bash}

~~~
$ for infile in *_1.fastq.gz
> do
>   base=$(basename ${infile} _1.fastq.gz)
>   trimmomatic PE ${infile} ${base}_2.fastq.gz \
>                ${base}_1.trim.fastq.gz ${base}_1un.trim.fastq.gz \
>                ${base}_2.trim.fastq.gz ${base}_2un.trim.fastq.gz \
>                SLIDINGWINDOW:4:20 MINLEN:25 ILLUMINACLIP:NexteraPE-PE.fa:2:40:15
> done
~~~
{: .bash}


Go ahead and run the for loop. It should take a few minutes for
Trimmomatic to run for each of our six input files. Once it's done
running, take a look at your directory contents. You'll notice that even though we ran Trimmomatic on file `SRR2589044` before running the for loop, there is only one set of files for it. Because we matched the ending `_1.fastq.gz`, we re-ran Trimmomatic on this file, overwriting our first results. That's ok, but it's good to be aware that it happened.

~~~
$ ls
~~~
{: .bash}

~~~
NexteraPE-PE.fa               SRR2584866_1.fastq.gz         SRR2589044_1.trim.fastq.gz
SRR2584863_1.fastq.gz         SRR2584866_1.trim.fastq.gz    SRR2589044_1un.trim.fastq.gz
SRR2584863_1.trim.fastq.gz    SRR2584866_1un.trim.fastq.gz  SRR2589044_2.fastq.gz
SRR2584863_1un.trim.fastq.gz  SRR2584866_2.fastq.gz         SRR2589044_2.trim.fastq.gz
SRR2584863_2.fastq.gz         SRR2584866_2.trim.fastq.gz    SRR2589044_2un.trim.fastq.gz
SRR2584863_2.trim.fastq.gz    SRR2584866_2un.trim.fastq.gz
SRR2584863_2un.trim.fastq.gz  SRR2589044_1.fastq.gz
~~~
{: .output}

> ## Exercise
> We trimmed our fastq files with Nextera adapters,
> but there are other adapters that are commonly used.
> What other adapter files came with Trimmomatic?
>
>
>> ## Solution
>> ~~~
>> $ ls ~/miniconda3/pkgs/trimmomatic-0.38-0/share/trimmomatic-0.38-0/adapters/
>> ~~~
>> {: .bash}
>>
>> ~~~
>> NexteraPE-PE.fa  TruSeq2-SE.fa    TruSeq3-PE.fa
>> TruSeq2-PE.fa    TruSeq3-PE-2.fa  TruSeq3-SE.fa
>> ~~~
>> {: .output}
>>
> {: .solution}
{: .challenge}

We've now completed the trimming and filtering steps of our quality
control process! Before we move on, let's move our trimmed FASTQ files
to a new subdirectory within our `data/` directory.

~~~
$ cd ~/dc_workshop/data/untrimmed_fastq
$ mkdir ../trimmed_fastq
$ mv *.trim* ../trimmed_fastq
$ cd ../trimmed_fastq
$ ls
~~~
{: .bash}

~~~
SRR2584863_1.trim.fastq.gz    SRR2584866_1.trim.fastq.gz    SRR2589044_1.trim.fastq.gz
SRR2584863_1un.trim.fastq.gz  SRR2584866_1un.trim.fastq.gz  SRR2589044_1un.trim.fastq.gz
SRR2584863_2.trim.fastq.gz    SRR2584866_2.trim.fastq.gz    SRR2589044_2.trim.fastq.gz
SRR2584863_2un.trim.fastq.gz  SRR2584866_2un.trim.fastq.gz  SRR2589044_2un.trim.fastq.gz
~~~
{: .output}

> ## Bonus Exercise (Advanced)
>
> Now that our samples have gone through quality control, they should perform
> better on the quality tests run by FastQC. Go ahead and re-run
> FastQC on your trimmed FASTQ files and visualize the HTML files
> to see whether your per base sequence quality is higher after
> trimming.
>
>> ## Solution
>>
>> In your AWS terminal window do:
>>
>> ~~~
>> $ fastqc ~/dc_workshop/data/trimmed_fastq/*.fastq*
>> ~~~
>> {: .bash}
>>
>> In a new tab in your terminal do:
>>
>> ~~~
>> $ mkdir ~/Desktop/fastqc_html/trimmed
>> $ scp dcuser@ec2-34-203-203-131.compute-1.amazonaws.com:~/dc_workshop/data/trimmed_fastq/*.html ~/Desktop/fastqc_html/trimmed
>> $ open ~/Desktop/fastqc_html/trimmed/*.html
>> ~~~
>> {: .bash}
>>
>> Remember to replace everything between the `@` and `:` in your scp
>> command with your AWS instance number.
>>
>> After trimming and filtering, our overall quality is much higher,
>> we have a distribution of sequence lengths, and more samples pass
>> adapter content. However, quality trimming is not perfect, and some
>> programs are better at removing some sequences than others. Because our
>> sequences still contain 3' adapters, it could be important to explore
>> other trimming tools like [cutadapt](http://cutadapt.readthedocs.io/en/stable/) to remove these, depending on your
>> downstream application. Trimmomatic did pretty well though, and its performance
>> is good enough for our workflow.
> {: .solution}
{: .challenge}-->
