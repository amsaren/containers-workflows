---
topic: bioapplications
title: Tutorial1 - Using existing Singularity images
---

# Using existing Singularity images

💬 In this tutorial we get a bit deeper with using Singularity containers. 

To run these exercises you need to container image `tutorial.sif`. If you did not 
already download it in the previous tutorial, download it now:

```bash
    wget  https://a3s.fi/saren-2001659-pub/tutorial.sif
 ```

## Using files
💬 Singularity containers have their own internal file system that is separate from the host file system. 
- The internal file system is always read-only when the container is run with normal user rights.

💭 In most real-world use cases you probably want to access the host file system to read and write files. 
- To do this you must bind a writable directory on host file system to the internal file system of the container. 

💡 This is done with command line argument `--bind` (or `-B`). The basic syntax is 
`--bind /path/in/host:/path/inside/container`.

💭 Some remarks:  
➖ The bind path does not need to exist inside the container – it is created if necessary.  
➖ More than one bind pair can be specified.  
➖ The option is available for all the run methods described in the previous tutorial.  

1. To run these exercises in Puhti, use `sinteractive`:
    ```bash
    sinteractive --account project_xxxx   # Change the xxxx for the project number
    ```
2. Try listing the contents of your project directory (substitute the correct path) from inside the container without `bind`:
    ```bash
    export SCRATCH=/scratch/project_xxxx/yourcscusername    # Replace xxxx and yourcscusername
    singularity exec tutorial.sif ls $SCRATCH
    ```
3. The container can not see the host directory, so you will get a "No such file or directory" error.

4. Now try binding host directory `/scratch` to directory `/scratch` inside the container:
    ```bash
    singularity exec --bind $SCRATCH:/scratch tutorial.sif ls /scratch
    ```
    - Or:
    ```bash
    singularity exec --bind /scratch:/scratch tutorial.sif ls $SCRATCH
    ```
5. This time the host directory is linked to to the container directory and the command shows what the container sees inside `/scratch`.

💡 You can use `bind`to set the container, for example, to find input data or configuration files from a certain directory.

6. Bind host directory specified in `$SCRATCH` into directory `/input`:
```bash
singularity exec --bind $SCRATCH:/input tutorial.sif ls /input
```

### Using the singularity wrapper
1. If you use wrapper script `singularity_wrapper`, it will take care of the binds for most common use cases:
    ```bash
    singularity_wrapper exec tutorial.sif ls $SCRATCH
    ```
2. If environment variable `$SING_IMAGE` is set, you don't even need to provide path to the image file:
    ```bash
    export SING_IMAGE=$PWD/tutorial.sif
    singularity_wrapper exec ls $SCRATCH
    ```

‼️ Since some modules set `$SING_IMAGE` when loaded, it is a good idea to start with `module purge` or `unset SING_IMAGE`
if you plan to use it, to make sure correct image is used.

## Environment variables
💬 Some software may reguire some environment variables to be set, e.g. to point to some reference data or a configuration file.

💬 Most environment variables set on the host are inherited by the container. 
- Sometimes this may be undesirable. 
- With command line option `--cleanenv` host environment is not inherited by the container.

💬 To set an environment variable specifically inside the container, you can set an environment variable `$SINGULARITYENV_xxx` (where xxx is the variable name) on the host before invoking the container.

1. Set some test variables:
    ```bash
    export TEST1="value1"
    export SINGULARITYENV_TEST2="value2"
    ```
2. Compare the outputs of:
    ```bash
    env |grep TEST
    singularity exec tutorial.sif env |grep TEST
    singularity exec --cleanenv tutorial.sif env |grep TEST
    ```
    - The first command is run on host, and we see `$TEST1` and `$SINGULARITYENV_TEST2`. 
    - The second command is run in the container and we see `$TEST1` (inherited from host) and `$TEST2` (specifically set inside the container by setting `$SINGULARITYENV_TEST2` on host). 
    - The third command is also run  inside the container, but this time we omitted host environment variables, so we only see `$TEST2`.
3. Note that any variables on command line are substituted by their values on the host:
    ```bash
    singularity exec tutorial.sif echo $TEST1
    singularity exec tutorial.sif echo $TEST2
    ```
    - The first line prints the value set in the host
    - The second line will result in empty output because a variable called `$TEST2` has not been set on host. (It was `SINGULARITYENV_TEST2="value2"` remember?)

## Exploring containers

💬 Our test container includes program `hello2`, but it is not in the `$PATH`. 

1. One way to find it is to try running `find` inside the container:
    ```bash
    singularity exec tutorial.sif find / -type f -name "hello2" 2>/dev/null
    ```
2. You can now run it by providing the full path:
    ```bash
    singularity exec tutorial.sif /found/me/hello2
    ```
3. Or you could add it to `$PATH` inside the container:
    ```bash
    export SINGULARITYENV_PREPEND_PATH=/found/me
    singularity exec tutorial.sif hello2
    ```

💡 If you can't locate the desired binary with `find`, you can always use `singularity shell` to explore the container.

## Running Singularity as a batch job
💡 `Singularity exec` is the run method you will typically use in a batch job script.

1. Make a file called `test.sh`:
    ```bash
    module load nano   # The computing node does not have nano by default
    nano test.sh
    ```
2. Copy the following contents into the file and change "project_xxxx" to the correct project name:

    ```bash
   #!/bin/bash
   #SBATCH --job-name=test
   #SBATCH --account=project_xxxx
   #SBATCH --partition=test
   #SBATCH --time=00:01:00
   #SBATCH --mem=1G
   #SBATCH --ntasks=1
   #SBATCH --cpus-per-task=1

   singularity exec tutorial.sif hello_world
    ```
    
    💬 In nano you can use `ctrl + o` to save and `ctrl + x` to exit.

3. Submit the job to the queue with:
    ```bash
   sbatch test.sh
    ```

## Using a containerized application available as a module

Some software available in CSC has bee installed as containers. In this example we will
run one such application.

1. Download some test data
    ```bash
    wget https://a3s.fi/containers-workflows/data.tar.gz
    tar xf data.tar.gz
    ```
    This should get you two FASTQ files: `B1_sub_R1.fq` and `B1_sub_R2.fq`

2. Create a batch job script called `cutadapt.sh`: 
   ```bash
   nano cutadapt.sh
   ```

3. Copy the following contents into the file and change "project_xxxx" to the correct project name:
   
   ```bash
   #!/bin/bash
   #SBATCH --job-name=test
   #SBATCH --account=project_xxxx
   #SBATCH --partition=test
   #SBATCH --time=00:10:00
   #SBATCH --mem=1G
   #SBATCH --ntasks=1
   #SBATCH --cpus-per-task=1

   # Load the software module
   module load cutadapt/3.4

   # The module sets $SING_IMAGE, so we can omit image file in command
   singularity_wrapper exec cutadapt \
   -a ^GTGCCAGCMGCCGCGGTAA...ATTAGAWACCCBDGTAGTCC \
   -A ^GGACTACHVGGGTWTCTAAT...TTACCGCGGCKGCTGGCAC \
   -m 215 \
   -M 285 \
   --discard-untrimmed \
   -o B1_sub_R1_trimmed.fq \
   -p B1_sub_R2_trimmed.fq \
   B1_sub_R1.fq B1_sub_R2.fq

   ```

    💬 Note that in the above example the command line is quite long, so we have used escape charecter "\" to escape (i.e. ignore)
    new line characters. From the computer's point of view the whole singularity_wrapper command is just a single line.

    💬 This way of splitting the command can improve readability and make it easier to edit parameters, but one has to be careful with the 
    notation, or some of the command may be omitted. Make sure "\" precedes the newline directly. If there is a e.g. a space between
    the "\" and the newline, the space gets escaped, not the newline, and the line gets cut.

4. Submit the job to the queue with:
    ```bash
   sbatch cutadapt.sh
    ```

5. In this case the module also includes a wrapper script that allows us to run the program with just `cutadapt`.
Modify the batch job script so that you change `singularity_wrapper exec cutadapt` to `cutadapt` and re-submit. 
Does it still work?

6. You can take a look at the cutadapt wrapper script to see how it's done.

    First find out the path to the script:
    ```bash
    which cutadapt
     ```
    You can then take a look at it using the path:
    ```bash
    less /appl/soft/bio/cutadapt/bin/cutadapt
    ```

💡 For more information on batch jobs, please see [CSC Docs pages](https://docs.csc.fi/computing/running/getting-started/).

## More information

💬 This tutorial is meant as a brief introduction to get you started.

☝🏻 When searching the internet for instruction, pay attention that the instructions are for the same version of Singularity that you are using. There has been some command syntax changes etc. between the versions, so older instructions may not work with copy-paste.

💡 For authoritative instructions see [Singularity documentation](https://sylabs.io/docs/).
