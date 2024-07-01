# Genetic-Diversity-in-Spartina
Glimpse on the pipeline curated for this analysis. Research paper in the process.

What is a container?
A container image is a standard unit of software that contains everything that is needed to run an application quickly and reliably across different computing environments. From laptop, to workbench, to cloud, or HPC. Containers remove the issue of “it works on my computer”. The portability of containers increases collaboration. Containers are “read-only” environments which strengthens version control enhances scientific reproducibility.

We use two terms when we describe these containerized units of software:

Container which is the software unit when it is running, and Image which is used to describe the software when it is not running.

How to run a container on an HPC

We’ll use singularity (or Apptainer) to run our container on the HPC environment.

While Docker is one of the most popular tools for building and running containers, it’s not supported on most HPC systems. Docker builds images that can have root and user permissions and running docker containers can require a normal user to have sudo or root permissions…not something that is possible on an HPC system.

Singularity and podman are two alternatives that allow you to build and run container images without the need for root permissions. Singularity is the only container runtime engine available on the Discovery cluster.

Where do I get a container image?

You can download a container from a public repository, for example dockerhub, singularity hub have containers that you will need to pull to a cluster (or run on your laptop).

Here’s how you would download the seedpod container using singularity. This will create a file called seedpod_rstudio.sif

# get on a compute node
srun --constraint=ib --pty /bin/bash

# load the singularity module
module load singularity/3.10.3

# novigate to where you want to put the container image
cd /work/seedpod/container

# make a cache and tmp directory. Singularity will write here when it pulls the container. 
mkdir -p cache tmp
export SINGULARITY_CACHEDIR=$(pwd)/cache 
export SINGULARITY_TMPDIR=$(pwd)/tmp

# pull the container
singularity pull --docker-login docker://containers.rc.northeastern.edu/seedpod/seedpod:rstudio

Alternatively, on the Discovery cluster, we have a path on the filesystem where you can find a number of containers, including the seedpod container.

You’ll see the singularity format image (.sif) and a containerfile, which has the instructions for the container to be build by a container runtime engine (podman, docker, singularity). The containerfile thus has a list of all the packages that were installed in the image.


ls /shared/container_repository/bioinformatics
## seedpod_containerfile
## seedpod_rstudio.sif
Testing the seedpod container
You can follow the guide above to pull the container to a space that you prefer on the Discovery cluster or other cluster that you’re working on. If you’re already on the Discovery cluster follow these steps to open and run the container.

# get on a compute node
srun --pty /bin/bash

# load the singularity module
module load singularity/3.10.3

# run the container
singularity run /shared/container_repository/bioinformatics/seedpod_rstudio.sif

#this will change your prompt to

Singularity>
Explore the file system

When you’re in the container you can see the file system within the container. by default your home directory is also bound to the container and can be viewed.

Let’s test this by looking at some of the packages that are installed in the container and in your home

# Shows whats in your home
Singularity> ls

# Show what is in the /opt directory in the container
Singularity> ls /opt
We can test this further by looking for another directory system that you have access to, but that the container doesn’t “know” about.

# look in /scratch
# put your username in place of mine (or prepare for the shock of a very messy scratch)

Singularity> ls /scratch/s.caplins
There’s a way to bind a directory to the container so it can “see” the directory. We will use the -B or --bind flag.

The --bind or -B flag takes two parts separated by a colon, the part that lives in your filesystem, and what you want to call it in the container.

For example with this I could add the directory “/work:/play”. This means that my directory in /work will be called /play when I am in the container.

In order to run this example we’ll also need to bind the ‘/shared’ directory so we can run the container

# if you're still in the container exit out
Singularity> exit

# run singularity run now with /scratch and /work bound to the container
singularity run -B "/work:/play" /shared/container_repository/bioinformatics/seedpod_rstudio.sif

# and now see if you can see whats in the /play directory. Here you can give it the whole path to a space you have access to in Discovery

Singularity> ls /play/rc/s.caplins
In practice we recommend keeping directory names simple when binding them to the container. For example, this is how I would bind /scratch and /work to the seedpod container. You don’t need to give it the full path to a specific location in work or scratch. That can be done when you’re in the container, or in your sbatch script.

Let’s exit out of the container above where “/work” = “/play” and set the same name outside and inside the container.

singularity run -B "/work:/work,/scratch:/scratch" /shared/container_repository/bioinformatics/seedpod_rstudio.sif
Data download
Let’s download some data to work with. This is all data that came from Spartina alterniflora, we just made it smaller to fit the analyses in this tutorial.

# navigate to where you want to put the files
cd /work/rc/s.caplins

# make a directory if you wish
mkdir seedpod_tutorial

cd seedpod_tutorial

wget https://github.com/northeastern-rc/training_seedpod/raw/main/tutorial_data.tar.gz

# look at the files we downloaded
ls

# un-compress the data files
tar -xvzf tutorial_data.tar.gz
Spend a little bit exploring the files that we downloaded with ls.

FastQC
Time to actually do something! Let’s run fastqc to view the quality of our raw data. We first get on a compute node (skip if you’re already on one), and load the singulalrity module (also skip if you’ve done this above).

# skip these steps if you're still in the container and see the Singularity> prompt
srun --pty /bin/bash
module load singularity/3.10.3
singularity run -B "/work:/work,/scratch:/scratch" /shared/container_repository/bioinformatics/seedpod_rstudio.sif

#check that we can find the fastqc command
Singularity> fastqc -h

# run fastqc on a sample
Singularity> fastqc fastqc/*  -o fastqc

Great. Let’s check the results by confirming that the files are there with ls data/fastqc.

In the discovery cluster, you can view html files using the app HTML Viewer.

Angsd
We can use the program angsd on bam files to get genotype likelhoods. Here we will do this for a subset of samples.

Bam files are alignment files, from mapping a sample to a reference genome, and are in a binary format (bam = binary sequence alignment file). We created the .bam files in ipyrad and will show you how at the end of this tutorial.

Again make sure you’re on a compute node and have bound the /scratch and /work directories to the container.

There are a few parameters you can set when estimating genotype likelihoods. You can see the manual for angsd for a full description of these parameters.

Before we jump in, we will make one of the inputs that angsd needs, which is a list of the bam files.

We’ll do this with some bash scripting and our favorite command ls.


# cd to where the bams directory is
Singularity> cd data/angsd

# ls by itself gives us a list of the bams
Singularity> ls bams/

# with some additions we can get the full path. the | (pipe) just shows the head or first couple of lines
Singularity> ls -d -1 bams/* | head

# now we'll use the > to write the output of ls to a file
Singularity> ls -d -1 bams/* > bam.filelist
Now let’s use that bam.filelist to generate genotype likelihoods in angsd.

# then run angsd to check that the program is there

Singularity> angsd

# and then the full line to generate genotype likelihoods

Singularity> angsd -bam bam.filelist -GL 2 -out genotype_likelihoods -doMajorMinor 1  -doGlf 2 -doMaf 2 -SNP_pval 1e-2 
PCangsd
Now we can use those genotype likelihood estimates to generate a principal component matrix.

# navigate to the pcangsd directory
Singularity> cd data/pcangsd

Singularity> pcangsd -b ../pcangsd/genotype_likelihoods.beagle.gz -o pca_out -t 28
R
We can visualize the output of pcangsd in R. It’s a bit complicated to get rstudio running from the container, so we’ll use base R. But there is rstudio in the seedpod container.

This code will take the pcaangsd covariance matrix and calculate eigen vales and then make a plot. The plot will be output as a .pdf, which we can view with the files app on the OOD.

# navigate to where the R script is
Singularity> cd data/R

# use the command 'Rscript' to run it
Singularity> Rscript pca_script.R
Ipyrad
We used the program Ipyrad to map the raw data files to the genome and call SNPs. The first part of running ipyrad requires generating a params file. This params file is generated by ipyrad and after being generated can be edited to modify parameters.

In the container, ipyrad was built in a conda env. So we will first activate the conda environment, and then tell ipyrad to generate a params file.

# to see the conda environments in the container

Singularity> conda env list

# activate the ipyrad environment

Singularity> source activate ipyrad

(ipyrad) Singularity>

# Create params file to give inputs and reference genome along with other parameters
  
(ipyrad) Singularity>  ipyrad -n spal # spal is name of the params file

# you should get something like this!!

  #(ipyrad) Singularity> ipyrad -n spal
  #New file 'params-spal.txt' created in /work/seedpod
  
Ipyrad has 7 steps in total 1. Sorting the raw data to organize it. 2. Filtering to remove low-quality reads. 3. Clustering similar sequences. 4&5. Generating consensus sequences from clusters. 6. Refining clusters of consensus sequences. 7. Formatting the final assembled data.


#on interaction node
(ipyrad) Singularity> ipyrad -p /work/seedpod/test/params-spal.txt -s 1234567 -c 32 -f  

# This is very long job, so we submitted sbatch 
#!/bin/bash
#SBATCH -N 1
#SBATCH -p short
#SBATCH -t 24:00:00
#SBATCH --exclusive
#SBATCH -n 1
#SBATCH --cpus-per-task=32
#SBATCH --mail-type=ALL
#SBATCH --mail-user=user@northeastern.edu
#SBATCH --output=/work/seedpod/test/spal_%j.out  # Log file for stdout
#SBATCH --error=/work/seedpod/test/spal_%j.err   # Log file for stderr
echo "Job started at: $(date)"

module load singularity/3.10.3

SINGULARITY_IMAGE="/shared/container_repository/bioinformatics/seedpod_rstudio.sif"
cd /work/seedpod/container

singularity exec -B "/work:/work" $SINGULARITY_IMAGE bash -c "
  set -e
  echo 'Activating Conda environment...'
  . /opt/miniconda/etc/profile.d/conda.sh
  conda activate ipyrad
  echo 'Environment activated.'
  
  echo 'Stopping any running ipcluster instances...'
  ipcluster stop --profile=profile_default || echo 'No ipcluster to stop.'
  echo 'Starting ipcluster...'
  ipcluster start --profile=profile_default --n 24 --daemonize
  sleep 10  # Give ipcluster some time to start
  
  echo 'Running ipyrad...'
  ipyrad -p /work/seedpod/test/params-spal.txt -s 7 -c 32 -f  
  
  echo 'Stopping ipcluster...'
  ipcluster stop --profile=profile_default
"

echo "Job finished at: $(date)"

  
