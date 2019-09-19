---
title: "Compute Canada Servers"
teaching: 30
exercises: 15
questions:
- "What is a Super Computer, or High Performance Computer (HPC)"
- "What makes it different form my not so super computer"
objectives:
- "Connect to a Compute Canada HPC"
- "Learn about the different HPC file system"
- "Search for a software, load it"
- "Submit a bash script to the the scheduler"
- "Look at the job status"
keypoints:
- "The scheduler send your script to a remote compute node where it is executed."
---

{% capture rapid %}def-poq-tr{% endcapture %}
{% capture reservation %}poq-reservation{% endcapture %}


# Connect to Béluga

Béluga is the main Compute Québec Super Computer and is one of the Computer Canada Federation HPC sytem along with Cedar, Graham and Niagara. You can find more information about these systems and documentation about how to use them on the [Compute Canada Wiki page](https://docs.computecanada.ca/wiki/Compute_Canada_Documentation)


If you do not already have a Compute Canada account, you can use the username and password provided to you at the beginning of this lesson.  

~~~
ssh -l <USER_NAME> beluga.calculcanada.ca
~~~
{: .bash}


You are now have a session on one of Béluga login node. It should look something like this:


~~~
###############################################################################
  _       __ _                   
 | |__   /_/| |_   _  __ _  __ _   Bienvenue sur Béluga / Welcome to Béluga
 | '_ \ / _ \ | | | |/ _` |/ _` |    
 | |_) |  __/ | |_| | (_| | (_| |  Aide/Support:    support@calculcanada.ca
 |_.__/ \___|_|\__,_|\__, |\__,_|  Globus endpoint: computecanada#beluga-dtn
                     |___/         Documentation:   docs.calculcanada.ca

###############################################################################
2019-00-00 Annonces en français
2019-00-00 Information to the public in English
[poq@beluga4 ~]$
~~~
{: .bash}



<!-- # The File system


There are more then one file systems mounted on Compute Canada Cluster.

~~~
/home
/project
/scratch
/localscratch
~~~
-->

# Installed software

You can see all software installed with the `module` command. To get a list of all software package, type `module avail`, type `module spider <string>` search if a module exist or to get information about a it.

You can the load the module using `module load`.

Lets do the procedure with a software that we will use later in the lesson, fastqc


~~~
 $ module spider fastq  

 ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
   fastq_screen:
 ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
     Description:
       FastQ Screen allows you to screen a library of sequences in FastQ format against a set of sequence databases so you can see if the composition of the library matches with
       what you expect.

      Versions:
         fastq_screen/0.11.4

 ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
   For detailed information about a specific "fastq_screen" module (including how to load the modules) use the module's full name.
   For example:

      $ module spider fastq_screen/0.11.4
 ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

 ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
   fastqc:
 ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
     Description:
       FastQC is a quality control application for high throughput sequence data. It reads in sequence data in a variety of formats and can either provide an interactive
       application to review the results of several different QC checks, or create an HTML based report which can be integrated into a pipeline.

      Versions:
         fastqc/0.11.5
         fastqc/0.11.8

 ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
   For detailed information about a specific "fastqc" module (including how to load the modules) use the module's full name.
   For example:

      $ module spider fastqc/0.11.8
 ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------


   Versions:
   fastqc/0.11.5  
   fastqc/0.11.8
~~~
{: .bash}  

Now with the exact string
~~~
$ module spider fastqc

-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
  fastqc:
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
    Description:
      FastQC is a quality control application for high throughput sequence data. It reads in sequence data in a variety of formats and can either provide an interactive application to
      review the results of several different QC checks, or create an HTML based report which can be integrated into a pipeline.

     Versions:
        fastqc/0.11.5
        fastqc/0.11.8

-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
  For detailed information about a specific "fastqc" module (including how to load the modules) use the module's full name.
  For example:

     $ module spider fastqc/0.11.8
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
~~~
{: .bash}  

With a version number,
~~~
$ module spider fastqc/0.11.8

-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
  fastqc: fastqc/0.11.8
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
    Description:
      FastQC is a quality control application for high throughput sequence data. It reads in sequence data in a variety of formats and can either provide an interactive application to
      review the results of several different QC checks, or create an HTML based report which can be integrated into a pipeline.

    Properties:
      Bioinformatic libraries/apps / Logiciels de bioinformatique

    You will need to load all module(s) on any one of the lines below before the "fastqc/0.11.8" module is available to load.

      nixpkgs/16.09

    Help:

      Description
      ===========
      FastQC is a quality control application for high throughput
      sequence data. It reads in sequence data in a variety of formats and can either
      provide an interactive application to review the results of several different
      QC checks, or create an HTML based report which can be integrated into a
      pipeline.


      More information
      ================
       - Homepage: http://www.bioinformatics.babraham.ac.uk/projects/fastqc/



~~~
{: .bash}  

There is a lot of info here, but one is stands out:  "You will need to load all module(s) on any one of the lines below before the "fastqc/0.11.8" module is available to load."

We are lucky here since nixpkgs/16.09 is loaded by default on and CC system, we ca see that by typing

~~~
$ module list

Currently Loaded Modules:
  1) nixpkgs/16.09   (S)      3) gcccore/.7.3.0  (H)   5) ifort/.2018.3.222 (H)   7) openmpi/3.1.2 (m)
  2) imkl/2018.3.222 (math)   4) icc/.2018.3.222 (H)   6) intel/2018.3      (t)   8) StdEnv/2018.3 (S)

  Where:
   S:     Module is Sticky, requires --force to unload or purge
   m:     MPI implementations / Implémentations MPI
   math:  Mathematical libraries / Bibliothèques mathématiques
   t:     Tools for development / Outils de développement
   H:                Hidden Module


~~~
{: .bash}

It is at number 1).


This mean that we can load `fastqc/0.11.8` without having to load dependencies:

~~~
$ module load fastqc/0.11.8
~~~
{: .bash}



# The SLURM Scheduler

### squeue

You are note supposed to run code on the login nodes. There are hundreds of compute node that are there to do that. The access to these node is taken cared of by a scheduler. All Compute Canada systems are using the [SLURM scheduler](https://www.schedmd.com).


Right now there are hundreds jobs running on the compute nodes, while more are waiting in queue to be executed. The `squeue` command lets you see all these jobs.

Lets type pipe `squeue` into `less` to see what is in the queue:

~~~
$ squeue | less
~~~
{: .bash}

To exit less, just type `q`

### sbatch


To send you own job in the queue, you will use the `sbatch` command.

While `sbatch` can take many options we only Compute Canada sytems onle need one to succesfully submit a script to the scheduler, the --account option. For this workshop, we will all use an accout epecually created for us `{{ rapid }}`, and use the command like this:

~~~
sbatch --account {{ rapid }} myscript.sh
~~~
{: .bash}


Will will see how that work in the next section.

![HPC](../img/detailed_super_computer.png)



# Run you first script on a Super computer


Use nano to create the following `hostname.sh` script:


~~~
#!/bin/bash
echo Hello from $HOSTNAME
T=5
echo sleeping $T seconds
sleep $T
~~~
{: .bash}

Than make that file executable:

~~~
chmod 755
~~~
{: .bash}

> ##  Exercise
>
> 1 Run the script on the login node, what is the output?  
> 2 Use the `sbatch` command to resubmit the script.  
>   - What job ID where you given?  
>  
> 3 Look at the status of you job using `squeue -u $USER`.  
>   - What is the meaning of the information contained in the `squeue` output?      
>
> 4 Where is the output of your job stored?  
>   - Hint look for a new file in the current directory
>   - What is in that file?
>
>> ## Solution
>>  1
>> ~~~
>>$ ./hostname.sh
>>Hello from beluga?.int.ets1.calculquebec.ca
>>sleeping 5 seconds
>>~~~
>>2
>> ~~~
>>$ sbatch ./hostname.sh
>>Submitted batch job <JOBID>
>>~~~
>>3
>> ~~~
>>          JOBID     USER      ACCOUNT           NAME  ST  TIME_LEFT NODES CPUS       GRES MIN_MEM NODELIST (REASON)
>>        <JOBID>      poq def-poq-ab_c    hostname.sh  PD    1:00:00     1    1     (null)    256M  (Priority)
>>~~~
4 Look for the following file
>>~~~
>> slurm-<JOBID>.out
>>~~~
>>{: .outputs}
> {: .solution}
{: .challenge}
