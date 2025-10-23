# Dardel for dummies
How to use Dardel PDC at KTH and other tools for programming.

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
ssh -i ~/.ssh/keyname username@servername.se
```

## Running Code on Dardel PDC

### Resource Allocation
```bash
# Standard allocation 
salloc -A naiss2024-22-projectnumber -p main --nodes=1 --time=02:00:00

# GPU allocation
salloc -A naiss2024-22-projectnumber -p gpu --nodes=1 --gpus=1 --time=02:00:00
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
If you load singularity, then you should run with (whatever this means) 
With this one, you also dont need to have an environment with Torch as is pre installed in the singularity. 
```bash
singularity exec --rocm -B /cfs/klemming /pdc/software/resources/sing_hub/rocm5.7_ubuntu22.04_py3.10_pytorch_2.0.1 python3 main.py
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

### Checking a job status
```bash
# Check all your jobs in the queue
squeue -u $USER

# Show detailed information about a specific job
scontrol show job <job_id>
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
df -i /cfs/klemming/home/a/username || df -i .

# Check which directories take most inodes:
find /cfs/klemming/home/a/username -xdev -type f -printf '%h\n' \
  | sort | uniq -c | sort -nr | head -n 50

# To free up, is safe (as long as I  understood right)
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

  Note: projectname is usually naiss2024-22-projectnumber

  
  



## Run in Background 

### Basic Bash Script Template
```bash
#!/bin/bash
```

### Running with nohup
```bash
# Run Python script in background
nohup python3 -u main.py >nohup_main.out &

# Check output
tail -f nohup_name.out
```

### Loop Through Variables
```bash
list=(1 2 3)
t=2
for k in "${list[@]}"
do
    file_path="nome_${k}_.npy"
    python3 main.py $k $t
done
```

## File Transfer
Using the right node to move big files (dardel-ftn01.pdc.kth.se). 
### Server to Local
```bash
# Move folders
scp -r username@dardel-ftn01.pdc.kth.se:/cfs/klemming/home/a/aurorap/folderpath /Users/aurorap/Desktop/folderpath

# Move single file
scp username@dardel-ftn01.pdc.kth.se:/cfs/klemming/home/a/aurorap/filepath/namefile /Users/aurorap/Desktop/filepath/namefile

# Move file using VPN
scp -i ~/.ssh/ed25519_vpn username@dardel-ftn01.pdc.kth.se:/cfs/klemming/home/a/aurorap/filepath/filename /Users/aurorap/Desktop/filepath/filename
```

### Local to Server
```bash
# Move folders
scp -r /Users/aurorap/Desktop/folderpath username@dardel-ftn01.pdc.kth.se:/cfs/klemming/home/a/aurorap/folderpath

# Move single file
scp /Users/aurorap/Desktop/filepath/namefile username@dardel-ftn01.pdc.kth.se:/cfs/klemming/home/a/aurorap/filepath/namefile

# Move file using VPN
scp -i ~/.ssh/ed25519_vpn /Users/aurorap/Desktop/filepath/filename username@dardel-ftn01.pdc.kth.se:/cfs/klemming/home/a/aurorap/filepath/filename
```


## Screen Sessions (Other Servers)

Screen allows running code while closing terminal and laptop, same as nohup.

### Basic Screen Commands
```bash
# Enter server
ssh username@servername.se

# Start screen
screen

# Start Python
python namefile.py

# Exit screen (detach)
# Press: Ctrl+A then D

# Return to screen
screen -r IDscreen

# Check running screens
screen -ls

# Force detach if attached
screen -d -r IDscreen

# Quit screen
# Press: Ctrl+A then type :quit

# Exit server
exit
```

### Jupyter Notebook Execution
```bash
# Run all Jupyter cells (doesn't always work)
jupyter nbconvert --to notebook --inplace --execute mynotebook.ipynb
```

## Python Parallel Processing

### Basic Setup
```python
import multiprocessing as mp

# Detect CPU cores
nprocs = mp.cpu_count()
print(f"Number of CPU cores: {nprocs}")

# Create pool
pool = mp.Pool(processes=nprocs)
```

### Simple Function (Single Input)
```python
def square(x):
    return x*x

# In main
result = pool.map(square, range(20))
print(result)
```

### Complex Function (Multiple Inputs)
```python
def power_n(x, n):
    return x**n

# In main
result = pool.starmap(power_n, [(x, 2) for x in range(20)])
print(result)
```

### Asynchronous Processing
```python
def power_n_list(x_list, n):
    return [x ** n for x in x_list]

def slice_data(data, nprocs):
    aver, res = divmod(len(data), nprocs)
    nums = []
    for proc in range(nprocs):
        if proc < res:
            nums.append(aver + 1)
        else:
            nums.append(aver)
    count = 0
    slices = []
    for proc in range(nprocs):
        slices.append(data[count: count+nums[proc]])
        count += nums[proc]
    return slices

# In main
inp_lists = slice_data(range(20), nprocs)
multi_result = [pool.apply_async(power_n_list, (inp, 2)) for inp in inp_lists]
result = [x for p in multi_result for x in p.get()]
print(result)
```

### Documentation
- Main: https://docs.python.org/3/library/multiprocessing.html
- Tutorial: https://www.kth.se/blogs/pdc/2019/02/parallel-programming-in-python-multiprocessing-part-1/
- Additional: https://superfastpython.com/multiprocessing-pool-python/
- Guide: https://medium.com/@mehta.kavisha/different-methods-of-multiprocessing-in-python-70eb4009a990

## Python Code Profiling

Profiling helps identify bottlenecks in code.

### Full Script Profiling
```bash
# Install cProfile
pip install cProfile

# Analyze entire script
python -m cProfile main.py
```

### Line-by-Line Profiling
```bash
# Install line profiler
pip install line_profiler
```

```python
# In code
from line_profiler import profile

# Before functions write
@line_profiler.profile
def your_function():
    pass
```

```bash
# In terminal
kernprof -l -v main.py
```
