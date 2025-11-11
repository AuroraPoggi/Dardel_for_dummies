# Dardel for dummies
This is a Guide for KTH employees who use the PDC Center's Dardel high-performance computing resources.

## DISCLAIMER:
🤯 Necessary disclaimer, this guide was created by PhD students in mathematics with little knowledge on HPC systems and basic programming skills. 
With this guide, we mainly hope to achieve the following:
* Share our knowledge & experiences with Dardel and, hopefuly, save you valuable time that you can spend on what you love 🫂.
* Help other KTH researchers who are struggling to set up the environment to run their codes faster and in an optimal way.
* Generate constructive discussion that benefits everyone 💬.
* Any comment and suggestions are more than welcome, the ultimate goal is to help each other 🤝. 

## Create a project 
1. Check on https://supr.naiss.se/resource/ which resourcer are available. Note: If you are a PhD student you can only ask for NAISS Small Compute. 
2. Go to https://supr.naiss.se/ and access using your KTH account. 
3. Click on the chosen compute and on "Create New Proposal for NAISS Compute ...". 
4. Fill in all the required boxes. 

## SSH Setup and Access

### Creating SSH Keys
```bash
# Create an SSH key
ssh-keygen -t ed25519

# Display public key to add to PDC portal
cat ~/.ssh/id_rsa.pub
```

### PDC Portal Setup
1. Go to https://loginportal.pdc.kth.se
2. Click on "add a new key"
3. Insert SSH public key (from `cat ~/.ssh/id_rsa.pub`)
4. Modify the key name with what you prefer and extend to all IP addresses

### To access Dardel through terminal:
```bash
# Run 
ssh usernamename@servername.se
```

### VSCode SSH Configuration
1. Click on `><` in bottom left corner
2. Click on "Connect to Host"
3. Click on "+" Add -new SSH Host...
4. Modify config file, adding the following:
```
Host nameplace
    HostName dardel.pdc.kth.se
    IdentityFile ~/.ssh/keyname
    IdentitiesOnly yes
    User username
```

### Access with VPN
```bash
ssh -i ~/.ssh/keynameVPN username@servername.se
```

## Running Code on Dardel PDC

### Resource Allocation 
the following is to run on a full node, see node type at (https://support.pdc.kth.se/doc/run_jobs/job_scheduling/#how-jobs-are-scheduled) under Dardel compute nodes. 
```bash
# Standard allocation 
salloc -A naiss2025-22-projectnumber -p main --nodes=1 --time=02:00:00

# GPU allocation
salloc -A naiss2025-22-projectnumber -p gpu --nodes=1 --gpus=1 --time=02:00:00
```

Otherwise, you can also write a sbatch script as:
```bash
#!/bin/bash -l
#SBATCH -A naiss2025-22-projectnumber
#SBATCH -J namejob
#SBATCH -t 02:00:00
#SBATCH --partition=main
#SBATCH --open-mode=append
#SBATCH -e name_error.e
#SBATCH -o name_output.o

srun bash bashname.sh
```
If you want to do it for gpu:
```bash
#!/bin/bash -l
#SBATCH -A naiss2025-22-projectnumber
#SBATCH -J namejob
#SBATCH -p gpu
#SBATCH -t 02:00:00
#SBATCH --nodes=1
#SBATCH --ntasks-per-node=1
#SBATCH -e name_error.e
#SBATCH -o name_output.o

# Activate your env

ml PDCOLD/23.12
ml singularity/4.1.1-cpeGNU-23.12

# Singularity specification, exec or shell
singularity ...
```

In case you want to run CPU but dont plan to use all the nodes, then you can use the shared nodes (thin node):
```bash
#!/bin/bash -l
#SBATCH -A naiss2025-22-projectnumber
#SBATCH -J namejob
#SBATCH -t 02:00:00
#SBATCH -p shared
#SBATCH --ntasks=2
#SBATCH --cpus-per-task=2
#SBATCH --mem=40G
#SBATCH --open-mode=append
#SBATCH -e name_error.e
#SBATCH -o name_output.o

srun bash bashname.sh
```

### Starting Interactive Session
```bash
# Once allocation is granted
srun --pty bash
# You should see: base username@nodename
```

### GPU Setup (if needed)
```bash
srun --pty bash
module load PDCOLD/23.12
module load singularity/4.1.1-cpeGNU-23.12
```
If you load singularity(has PyTorch), then you should run first:
```bash
singularity exec --rocm -B /cfs/klemming /pdc/software/resources/sing_hub/rocm5.7_ubuntu22.04_py3.10_pytorch_2.0.1 python3 main.py
```
Note: with this one, you also dont need to have an environment with Torch as is pre installed in the singularity and it directly execute the Python script (exec command). 

Alternative, first you open singularity using shell then run the Python script:
```bash
singularity shell --rocm -B /cfs/klemming /pdc/software/resources/sing_hub/rocm6.3_ubuntu24.04_py3.12_pytorch_release_2.4.0
```
followed by, inside the sigularity: 
```bash
python main.py
```

In case you need Python packages that are not included in Singularity, like PyTorch Geometric, you first need to create an environment and install them.
You need to do the following steps: 
* In python script path: 
```bash
singularity shell   -B /cfs/klemming/projects/supr/naiss2025-22-memoryid/env:/env   --env PYTHONPATH=/env/lib/python3.12/site-packages   /pdc/software/resources/sing_hub/rocm6.3_ubuntu24.04_py3.12_pytorch_release_2.4.0 
```
* Inside singularity
```bash
PYTHONPATH=/ca2env/lib/python3.12/site-packages:$PYTHONPATH
```
* Then run python script:
```bash
python main.py 
```

In case you have some data that are in another folder, for example in project memory, then you do:
* 
```bash
singularity shell -B /cfs/klemming/projects/supr/naiss2025-22-memoryid:/cfs/klemming/projects/pathtodatafolder \
  /pdc/software/resources/sing_hub/rocm6.3_ubuntu24.04_py3.12_pytorch_release_2.4.0
```
* Then run python script:
```bash
python main.py 
```

In case you have both data and python packages needed in another environment (Note: BE CAREFUL the environment does not have to contain torch (if you are in torch singularity otherwise it will lead to problems) then you follow these steps:
* In terminal, at location where python script is:
```bash
singularity shell \
  -B /cfs/klemming/projects/supr/naiss2025-22-memoryid/env:/env \ #modify with your right piath
  -B /cfs/klemming/projects/supr/naiss2025-22-memoryid:/cfs/klemming/projects/pathtodatafolder \
  /pdc/software/resources/sing_hub/rocm6.3_ubuntu24.04_py3.12_pytorch_release_2.4.0
```
* Inside the Singularity, (so you will read...
  
```bash
>Singularity
```
)
  then you do the following (to import the libraries not in that container): 
```bash
PYTHONPATH=/myenv/lib/python3.12/site-packages:$PYTHONPATH
```
* Run python script:
```bash
python main.py
```


### Environment Activation
```bash
conda activate myenv
```

### Running Scripts in python or bash, as usual
```bash
python main.py
# or
bash test.sh
```

## Checking a job status
```bash
# Check all your jobs in the queue
squeue -u $USER

# Show detailed information about a specific job
scontrol show job <job_id>
sstat --jobs=<job_id>

# Show all details
squeue -u $USER -h -o "%18i %.200j %.2t %.10M %.6D %R"

# Show detailed information about a specific PAST job
sacct --jobs=<job_id>
# or to get run executed before midnight of current day
sacct --starttime=YYYY-MM-DD

# Show from which project the job was sent to the queue
scontrol show job JOBID | grep -oP 'Account=\K[^ ]+'
```
### Cancel a running or pending job
```bash
scancel <job_id>
```

## Storage Management
(https://support.pdc.kth.se/doc/data_management/klemming/#storage-areas)

### Check Available Space
```bash
# Check disk space
df -h .

# Check directory usage
du -h --max-depth=1 | sort -hr

# Check individual file sizes
ls -lh

# Check subdirectories of your project sing most space
du -sh /cfs/klemming/projects/snic/my_proj/* | sort -hr

# Check how much space each member of the project is using
find /cfs/klemming/projects/snic/my_proj -printf "%s %u\n" | awk '{arr[$2]+=$1} END {for (i in arr) {print arr[i],i}}' | numfmt --to=iec --suffix=B | sort -hr


# Sometimes you get error disk quita axceeded but instead is the inodes, to check:
df -i /cfs/klemming/home/u/username || df -i .

# Check which directories take most inodes:
find /cfs/klemming/home/u/username -xdev -type f -printf '%h\n' \
  | sort | uniq -c | sort -nr | head -n 50

# To free up, is safe to do (pip will re-download all packages)
python3 -m pip cache purge

```

### Storage types:
* HOME: path
  ```bash
  /cfs/klemming/home/u/username
  ```
  of size 25GB with backup, is the path of the login node. Only username can access the folders, however, the size is limited and data are available at least after 6 months after last allocayion ended.
* PROJECTS: path
```bash
/cfs/klemming/projects/snic/projectname
``` 
size depends on allocation and it does not have backup. All the data in a project directory will be deleted 3 months after the project ends.
* Scratch path 
```bash
 /cfs/klemming/scratch/u/username
```
  size ulimited. All the data in a project directory will be deleted 3 months after the project ends.

  Note: projectname is usually naiss2025-22-projectnumber


## File Transfer
Using the right node to move big files (dardel-ftn01.pdc.kth.se). 
### Server to Local
```bash
# Move folders
scp -r username@dardel-ftn01.pdc.kth.se:/cfs/klemming/home/u/username/folderpath /Users/username/folderpath

# Move single file
scp username@dardel-ftn01.pdc.kth.se:/cfs/klemming/home/u/username/filepath/namefile /Users/username/filepath/namefile

# Move file using VPN
scp -i ~/.ssh/keynameVPN username@dardel-ftn01.pdc.kth.se:/cfs/klemming/home/u/username/filepath/filename /Users/username/filepath/filename
```

### Local to Server
```bash
# Move folders
scp -r /Users/username/folderpath username@dardel-ftn01.pdc.kth.se:/cfs/klemming/home/u/username/folderpath

# Move single file
scp /Users/username/filepath/namefile username@dardel-ftn01.pdc.kth.se:/cfs/klemming/home/u/username/filepath/namefile

# Move file using VPN
scp -i ~/.ssh/keynameVPN /Users/username/filepath/filename username@dardel-ftn01.pdc.kth.se:/cfs/klemming/home/u/username/filepath/filename
```



