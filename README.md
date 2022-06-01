# Assembly Pipelines
Created by Dua Vang for Masters Thesis - May 2022; Last updated June 2022

Pipelines include de novo, hybrid, and consensus assemblies for Genome-enabled SynCom in maize. 
 

## de novo assembly using SPAdes
NOTE: Pipeline is compatible with only MacOS (M1 chip)

### Installations: 
- Download  SPAdes here: [http://cab.spbu.ru/software/spades/ 
]()	


- Choose	“Download SPAdes binaries for Mac”, unzip the folder with Archive Utility. Doesn’t require compiling
Manual included: [http://cab.spbu.ru/files/release3.14.1/manual.html#sec3.5 ]()


- Alternatively: 
	
```
$ cd <directory of your bin folder>
$ curl http://cab.spbu.ru/files/release3.15.1/SPAdes-3.15.1-Darwin.tar.gz -o SPAdes-3.15.1-Darwin.tar.gz
$ tar -zxf SPAdes-3.15.1-Darwin.tar.gz
$ cd SPAdes-3.15.1-Darwin/bin/

```

### Pipeline: 
1. If adapters are still on your reads, run ```trim_galore``` to remove them:
	
	```
	$ trim_galore --paired --length 80 RDH20_R1.fq RDH20_R2.fq
	
	```

2. Assemble using ```spades.py``` command

	```
	$ spades.py --pe1-1 ./RDH20_R1_val_1.fq --pe1-2 ./RDH20_R2_val_2.fq -o RDH20_SPAdes-1
	
	```
	```--pe1-1``` and ```--pe1-2``` specify the two paired end read files.      
```-o``` (required) makes a new output directory under this name


	The output file is called **_scaffolds.fasta_**


3. Submit fasta file into BLAST Microbial Genomes and narrow search to the genus of interest (this can take up to 2 hours to run, but also may finish in less than five minutes). File can be submitted to annotator program such as RAST or PATRIC to analyze contigs. 

**NOTE: SPAdes seems to work best for small genomes / higher coverage. With large genomes (and/or high GC content?), I’ve been getting thousands of contigs for Streptomyces genomes compared to less than 100 for many others.**


## Hybrid assembly using Unicycler
NOTE: Pipeline is compatible with any interface 

1. Create PATRIC profile: [https://www.patricbrc.org](https://www.patricbrc.org)
2. Under the "Services" tab, Click on **Comprehensive Genome Analysis**
3. Input your trimmed paired-end (Illumina) reads and single read library (Nanopore). 
4. Leave the assembly parameters on default settings. 
5. Label your strain ID and "submit" run. This process should take no longer than 5-10 minutes. 
6. Once finished, navigate to the folder containing the assembled genome and annotation files. Download the "Full Genome Report", "Genome Assembly Report" and "annotation.xls." 


## Concensus assembly using Trycycler
NOTE: Pipeline was created by Brea LaSarre and optimized by Ashley Paulsen. Pipeline is compatible with any interface. 

### Access hpc:

    ssh username@condo.its.iastate.edu

You have access to two directories, your home where you log in and the work directory for your group. 
Your home directory only has 5 GB of space, its mostly for batch files and temporary storage.
The group directory has far more space and is where you can install programs.

    /home/username/
    /work/LAS/larryh-lab/

You log into the head node, but try to only use that for basic tasks and logging in. To open an interactive session run 
    
    salloc -N1 -t 2:00:00

This requests 1 node for 2 hours (feel free to change the time). You can exit a node by calling ```exit```. We'll touch on batch modes later.

### Install miniconda3:
You can't module load miniconda, it won't work right. So we will install our own. Only do this once.

    module load gcc
    mkdir -p ~/miniconda3
    wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh -O ~/miniconda3/miniconda.sh
    bash ~/miniconda3/miniconda.sh -b -u -p ~/miniconda3
    rm -rf ~/miniconda3/miniconda.sh
    ~/miniconda3/bin/conda init bash
    ~/miniconda3/bin/conda init zsh
### Create conda environment and install packages:
We will run everything in a conda environment that has all the packages we need, this is way easier than trying to install everything we need.
I have a conda environment already made in ```/larryh-lab/consensus_assemblies``` but I am unsure if it will work for other people, we'll have to test this.
To make your own, run the following code.

Go to the working directory and create the environment

    cd /work/LAS/larryh-lab/consensus_assembly/
    conda create --prefix ./trycycler-env

Install the packages you need
    
    conda config --add channels defaults
    conda config --add channels bioconda
    conda config --add channels conda-forge

    source activate trycycler-env/

    conda install trycycler filtlong any2fasta 
    conda install flye raven-assembler miniasm minipolish

Now you should have an environment where you can run everything. The environment I currently use is ```trycycler-env-2/```.

### Getting data to and from lss:

To get data to and from the lss you should use the data transfer node. You can run it while logged into the headnode.

    ssh username@condodtn.its.iastate.edu
    kinit
    #put in your password when asked

You can now copy files between condo and lss, but you need to know the file path. For example, the below code copies a file from lss to condo and a directory from condo to lss.

    cp /lss/research/larryh-lab/Ashley-P/Trycycler/reads.fastq /work/LAS/larryh-lab/consensus_assembly/

    cp -r /work/LAS/larryh-lab/consensus_assembly/S18_assembly/ /lss/research/larryh-lab/Ashley-P/Trycycler/


### Setting up your reads:

Activate your environment and your directory and navigate to the folder you have your reads

    source activate trycycler-env/
    cd Strain_ID

### Quality control:

    filtlong --min_length 1000 --keep_percent 95 input_reads.fastq.gz > QC_reads.fastq

### Subsample reads:

    trycycler subsample --reads QC_reads.fastq --out_dir read_subsets --count 8 --genome_size 5.2m
    
Trycycler starts by subsampling the reads and saving those subsets in a new directory. You can change the number of subsets with ```--count``` and the genome size is optional, but it will speed up your assembly if you have it.

Genome size may change depending on what strain you are assembling. 

You should end up with a directory ```read_subsets``` that contains 8 files named ```sample_*.fastq```.

### Assemblies and Batches:

Now you need to generate your assemblies. First you need a directory for them to go into.

 Raven and Miniasm/Minipolish can be run quickly in a compute node, but Flye needs a lot of RAM and should be run in a fat node. So, we are going to talk about Slurm and batch mode. 

Hpc uses Slurm to allocate resources. When you call ```salloc```, you are requesting a node to use interactively. You can also run things in batches. There is a slurm script generator for condo available at [hpc.iastate.edu](https://www.hpc.iastate.edu/guides/condo-2017/slurm-job-script-generator-for-condo) but I will provide one here for example. To use this you create a .txt file with the slurm script first, followed by your code.

    #!/bin/bash

    # Copy/paste this job script into a text file and submit with the command:
    #    sbatch thefilename
    # job standard output will go to the file slurm-%j.out (where %j is the job ID)

    #SBATCH --time=2:00:00   # walltime limit (HH:MM:SS)
    #SBATCH --nodes=1   # number of nodes
    #SBATCH --ntasks-per-node=32   # 32 processor core(s) per node 
    #SBATCH --partition=fat    # fat node(s)
    #SBATCH --job-name="all-assemblies"
    #SBATCH --mail-user=ashleyp1@iastate.edu   # email address
    #SBATCH --mail-type=BEGIN
    #SBATCH --mail-type=END
    #SBATCH --mail-type=FAIL

    # LOAD MODULES, INSERT CODE, AND RUN YOUR PROGRAMS HERE

    # Runs flye, raven, and miniasm assemblies
    # just change the folder and genome size

    #loads the compiler
    module load gcc

    #change directory to dir containing conda env
    cd /work/LAS/larryh-lab/consensus_assemblies/

    #activate conda env
    source activate trycycler-conda-env-2/

    #change this folder to your strain
    cd Strain_ID
    
    #make sure to comment this command out if running multiple times, you only want it to run once
    mkdir assemblies

    #change genome size
    #threads is based on the node
    threads=32
    genome_size=5.5m

    #flye assembly
    flye --nano-raw read_subsets/sample_01.fastq --threads "$threads" --genome-size "$genome_size" --out-dir assembly_01
    
    #save the fasta and delete the rest
    cp assembly_01/assembly.fasta assemblies/assembly_01_flye.fasta
    rm -r assembly_01

    #miniasm and minipolish assembly
    minimap2 -t "$threads" -x ava-ont read_subsets
    sample_01.fastq read_subsets/sample_02.fastq > overlaps.paf
    miniasm -f read_subsets/sample_02.fastq overlaps.paf > assembly.gfa
    minipolish -t "$threads" read_subsets/sample_02.fastq assembly.gfa > polished.gfa

    #convert the miniasm assembly to fasta and save
    any2fasta polished.gfa > assemblies/assembly_02_miniasm.fasta

    #remove unwanted files
    rm polished.gfa
    rm overlaps.paf
    rm assembly.gfa

    #raven assembly
    raven --threads "$threads" read_subsets/sample_03.fastq > assemblies/assembly_03_raven.fasta
    
    #remove unwanted files
    rm raven.cereal

Lets say I saved this as ```batch-all-assemblies.txt``` After running this there should be a directory named ```Strain_ID/assemblies/``` that contains 3 fasta assemblies. To run this file you go to the directory you saved it in, it doesn't matter but I usually save mine in my home directory, and run

    sbatch batch-all-assemblies.txt

Make sure you are in the head node, not an interactive node, when you run this. You will get an email when the job starts to run, if it fails, and when its finished. For the full run you want assemblies for each of your subsets, so the batch file would be much longer. I have a master with all of the assemblies saved.

### Next Denovo:

You have to install Next Denovo from their .tar file and run everything in that file. I repeat, everything must be in the same file! I have a Next Denovo folder in the work directory that contains everything. Make a file within that folder for your assembly and copy the reads that you want to use into that folder. Next Denovo requires an input file and a run config file.

To make the input file is easy, but you must be in the right directory. Lets say I made a directory, ```A_assembly``` in the NextDenovo directory and I copied ```sample_01.fastq``` into it.
    
    ls sample_01.fastq > input.fofn

This makes the input file. The run config file is a text file similar to those used for slurm scripts. It has to saved as a .cfg but the name can be changed. I found its easiest to copy a run file from a previous assembly and to chang it using vim in the command line. You can edit it in a text editor if you're more comfortable with that. Below is a file I have used, ```1_run.cfg```, and the only thing I change is the workdir and the genome size. The workdir will create a directory ```1_rundir``` with the current directory and put all of the assembly files there.

    [General]
    job_type = local
    job_prefix = nextDenovo
    task = all # 'all', 'correct', 'assemble'
    rewrite = yes # yes/no
    deltmp = yes
    rerun = 3
    parallel_jobs = 2
    input_type = raw
    read_type = ont
    input_fofn = ./input.fofn
    workdir = ./1_rundir

    [correct_option]
    read_cutoff = 1k
    genome_size = 3.2m
    pa_correction = 2
    sort_options = -m 1g -t 2
    minimap2_options_raw =  -t 8
    correction_options = -p 15

    [assemble_option]
    minimap2_options_cns =  -t 8
    nextgraph_options = -a 1

Once you have the input file and config file you can run the assembly. 

    NextDenovo
        A_assembly
            sample_01.fastq
            input.fofn
            1_run.cfg

Check to make sure you have all of these files in the right location. You should be in the ```NextDenovo``` directory when you run the following command.

    python nextDenovo A_assembly/1_run.cfg

After it has run there should be a new directory ```1_rundir``` in the ```A_assembly``` directory that contains all of your assembly files. The final fasta file that trycycler needs is located in ```1_rundir/03.ctg_graph/nd.asm.fasta```. You can copy this to your assemblies folder with all of your other assemblies.

Running Next Denovo as a batch is possible, but a little more complicated. You need to make the assembly directory in the Next Denovo folder and make all of the ```run.cfg``` files before you can run the slurm script. Below is an example to run one Next Denovo assembly on the read subset ```sample_04.fasta```, copy the final assembly to the relevant folder, and then clean up so the next Next Denovo assembly can be run. 

    cd /work/LAS/larryh-lab/consensus_assemblies

    source activate trycycler-conda-env-2/

    #copy the read file to the NextDenovo dir
    cp /work/LAS/larryh-lab/consensus_assemblies/S18_bacillus/read_subsets/sample_04.fastq /work/LAS/larryh-lab/consensus_assemblies/NextDenovo/S18_assembly/

    #make the input file
    cd NextDenovo/S18_assembly/
    ls sample_04.fastq > input.fofn
    cd ..

    #assemble
    python nextDenovo S18_assembly/4_run.cfg

    #copy the final assembly back to my trycycler folder
    cp ./S18_assembly/4_rundir/03.ctg_graph/nd.asm.fasta /work/LAS/larryh-lab/consensus_assemblies/S18_bacillus/assemblies/assembly_04_nextDenovo.fasta
    
    # delete the input file so I can run the next subset
    rm S18_assembly/input.fofn

Next Denovo has trouble circularizing, so its final assemblies often has terminal repeats that are too long. You will have to manually fix this to use these assemblies. I run Circlator on my NextDenovo assemblies using the following commands. These are the settings recommended for nanopore reads and the inputs are the assembly, the reads, and the name of the output directory to generate. The final output will be labeled ```06.fixtstart.fasta``` in the output directory.

    circlator all --merge_min_id 85 --merge_breaklen 1000 assembly_nextDenovo.fasta QC_reads.fastq nextDenovo_circlator


### Trycycler: Cluster

Once you have all of your assemblies you can move on to the actual trycycler assembly. The first step is to cluster the contigs from the assemblies based on similarity. The following command does that.

    trycycler cluster --assemblies assemblies/*.fasta --reads QC_reads.fastq --out_dir trycycler

This will create a directory named trycycler. Within this directory will be a folder for each cluster and two files, ```contigs.phylip``` and ```contigs.newick```. The newick file is the important one for the moment. You will need to copy that file to somewhere you can access it and open it in your preferred tree viewer, I like Dendroscope. Now you need to decide which clusters are good and which are bad. There is an in depth explanation with examples in the [trycycler tutorial](https://github.com/rrwick/Trycycler/wiki/Clustering-contigs) and I recommend that you read through them. 

Once you have decided which clusters to keep you can either delete or rename the ones you don't want. You can also go into the ``cluster_00*`` file and delete individual contigs, if you only want to remove one from a cluster. If you have to delete too many then you may need to go back and subsample again and generate more assemblies. 

### Trycycler: Reconciling clusters

Next you will reconcile your chosen clusters. This command has to be run on each cluster you want to keep. 

    #Reconcile each cluster
    trycycler reconcile --reads QC_reads.fastq --cluster_dir trycycler/cluster_001

You may have to manually intervene at this step as well. Reconcile won't run if the contigs have too much size variation or are too different from each other. You may have to go in and delete any contigs that are causing problems. Cycling through generating assemblies, clustering, and reconciling is the most time consuming and finicky step, it may take a few tries to get all of the clusters reconciled. 

I recommend running reconcile in an interactive session so you can adjust parameters as you go. For example, I often had to adjust the max add and max trim parameters by just a touch to enable circularisation. ```--max_add_seq``` and ```--max_add_seq_percent``` default to 1000 and 5, which means Trycycler will be willing to add up to 1000 bp or 5% of a contig's length (whichever is smaller) to circularise it. ```--max_trim_seq``` and ```--max_trim_seq_percent``` default to 50000 and 10 which means Trycycler will be willing to remove up to 50000 bp or 10% of a contig's length (whichever is smaller) to circularise it. For example, I may have one contig that requires 1234 bp to be added to circularise and another that needs 50654 trimmed. Both of these will cause reconcile to fail. You can either delete these contigs, or you can adjust the parameters, like so.

    trycycler reconcile --reads QC_reads.fastq --cluster_dir trycycler/cluster_001 --max_add_seq 1500 --max_trim_seq 51000

This will allow both contigs to be included. You can also choose to just include one and not the other.
    
### Trycycler: Multiple sequence alignment, partitioning reads, and generating a consensus

The last steps are quick and easy and don't require much, if any, manual intervention. You have to perform a multiple sequence alignment on each of your reconciled clusters. Then you partition your reads among these clusters, this command is just run once and covers all the clusters so make sure you have deleted or renamed your bad ones. Finally, you generate the final consensus for each cluster.

    #Align each cluster
    trycycler msa --cluster_dir trycycler/cluster_001
    
    #Partition reads among all clusters
    #delete or rename bad clusters
    trycycler partition --reads QC_reads.fastq --cluster_dirs trycycler/cluster_*
    
    #Find a consensus for each cluster
    trycycler consensus --cluster_dir trycycler/cluster_001

### Polishing

#### Polishing with long reads

First you polish with your long reads, using racon and medaka. Racon should always be done before medaka, because medaka was trained to run on racon polished reads. I do two rounds of racon and then two rounds of medaka for each of the clusters. You could increase the rounds, but you risk introducing errors with more rounds.

I tried to get racon and medaka on the same conda environment but they ended up conflicting, so make sure you change the environment when you switch between them.

Racon

    conda create --prefix ./racon-env
    source activate racon-env/
    conda install racon

    mkdir racon
    #copy the final assembly
    cp ./trycycler/cluster_001/7*.fasta ./racon

    #first racon round
    bwa index racon/7_final_consensus.fasta
    
    bwa mem -t 32 -x ont2d racon/7_final_consensus.fasta QC_reads.fastq > racon/mapping1.sam
    
    racon -m 8 -x -6 -g -8 -w 500 -t 32 QC_reads.fastq racon/mapping1.sam racon/7_final_consensus.fasta > racon/racon1.fasta

    #second racon round
    bwa index racon/racon1.fasta
    
    bwa mem -t 32 -x ont2d racon/racon1.fasta QC_reads.fastq > racon/mapping2.sam
    
    racon -m 8 -x -6 -g -8 -w 500 -t 32 QC_reads.fastq racon/mapping2.sam racon/racon1.fasta > racon/racon2.fasta
    cd ..

    conda deactivate

Medaka
    conda create --prefix ./medaka-env
    source activate medaka-env/
    conda install medaka

    #first round of medaka
    medaka_consensus -i QC_reads.fastq -d racon/racon2.fasta -o racon_medaka_1 -t 32

    #second round of medaka
    medaka_consensus -i QC_reads.fastq -d racon_medaka_1/consensus.fasta -o racon_medaka_2 -t 32

Remember, these polishing steps need to be performed on each of your final clusters.

### Polishing with short reads:

The assemblies are now polished with short reads using POLCA. POLCA is a polishing tool that is part of the MaSuRCA hybrid assembler, but it can be used on its own.

First, you must combine your polished consensus assemblies into one file. You can skip this step if you only have 1 cluster, otherwise use the cat command to make a combined file, an example of which is shown below.

    cat polish_1.fasta polish_2.fasta > combined.fasta

Once they're combined you can move onto polishing with POLCA.

    conda create --prefix ./masurca-env
    source activate masurca-env/
    conda install masurca

    mkdir polca

    #Quality control short reads
    fastp --in1 E05_illumina_R1.fastq --in2 E05_illumina_R2.fastq --out1 R1.fastq --out2 R2.fastq --unpaired1 unpaired1.fastq --unpaired2 unpaired2.fastq
    
    #-a is the assembly file to be polished
    polca.sh -a racon_medaka_2/consensus.fasta -r 'R1.fastq R2.fastq' -t 32 -m 1G
    
    #clean up files
    #consensus is based on the name of the fasta file
    mv ./consensus* ./polca/
    mv ./*.err ./polca/
    cp ./polca/consensus.fasta.PolcaCorrected.fa ./polished_consensus.fasta

### Assessing assembly quality:

#### QUAST
Don't bother with this.
Must be done in QUAST folder, copy the final assembly into the QUAST folder then run QUAST in the same folder. 

    source activate quast-env/
    cd quast-5.2.0
    python quast.py polished_consensus.fasta -o output_dir

### Hybrid consensus assembly:

#### Assemblers

For samples with low nanopore coverage you can skip the subsampling and instead run the long read assemblers and the hybrid assemblers, giving you up to 7 different assemblies.

Code to run Unicycler, Spades, and MaSuRCA hybrid assemblies and save the assemblies in the assemblies directory. There is another assembler, HASLR, but I have not been able to get it to run correctly. After you have run these you can follow the pipeline from clustering onwards.

    source activate hybrid-env/ #make a conda env with unicycler and spades, can use the masurca env for masurca

    cd R19_ensifer

    #change genome size
    threads=32
    genome_size=7.2m

    # Quality control illumina reads
    fastp --in1 R19_illumina_R1.fastq --in2 R19_illumina_R2.fastq --out1 R1.fastq.gz --out2 R2.fastq.gz --unpaired1 u.fastq.gz --unpaired2 u.fastq.gz

    # Run Unicycler
    unicycler -1 R1.fastq.gz -2 R2.fastq.gz -l QC_reads.fastq -o unicycler_assembly -t "$threads"
    cp unicycler_assembly/assembly.fasta assemblies/assembly_unicycler.fasta

    # Run Spades
    python ../hybrid-env/bin/spades.py -o spades_assembly -1 R1.fastq.gz -2 R2.fastq.gz --nanopore QC_reads.fastq -t "$threads"
    cp spades_assembly/contigs.fasta assemblies/assembly_spades.fasta

    # Run MaSuRCA
    module load numactl
    mkdir masurca_assembly
    cp QC_reads.fastq ./masurca_assembly/
    cp R1.fastq.gz ./masurca_assembly/
    cp R2.fastq.gz ./masurca_assembly/
    cd masurca_assembly

    /work/LAS/larryh-lab/consensus_assemblies/hybrid-env/bin/masurca -t "$threads" -i R1.fastq.gz,R2.fastq.gz -r QC_reads.fastq

    #the masurca assembly is  masurca_assembly/CA.mr.33.17.15.0.02/primary_genome.scf.fasta (or something similar)

MaSuRCA doesn't output into a nice neat directory and will generate a whole mess of files in whatever directory you run it in, hence the need to create a new directory with the necessary files and to run it there. 

Reconciling the hybrid assemblies tends to require more manual intervention because you can't afford to delete too many. I had to run Circlator on some because Trycycler couldn't circularise them. I also had to adjust the max add and trim parameters a number of times, because I was less willing to delete contigs. Examples of the code for that are in the section on reconciling clusters.

