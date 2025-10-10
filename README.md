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
4. Modify the key name and extend to all IP addresses

### VSCode SSH Configuration
1. Click on `><` in bottom left corner
2. Click on "Connect to Host"
3. Click on "+" Add new SSH Host
4. Modify config file:
```
Host nameplace
    HostName dardel.pdc.kth.se
    IdentityFile ~/.ssh/sissa_id_ed25519 
    IdentitiesOnly yes
    User aurorap
```

### Access with VPN (Mac VPN drops every 24 minutes)
```bash
ssh -i ~/.ssh/ed25519_vpn aurorap@dardel.pdc.kth.se
```

## Running Code on Dardel PDC

### Resource Allocation
```bash
# Standard allocation (Kateryna's project 1707, Matthieu's 1600)
salloc -A naiss2024-22-1707 -p main --nodes=1 --time=02:00:00

# GPU allocation
salloc -A naiss2024-22-1707 -p gpu --nodes=1 --gpus=1 --time=02:00:00
```

### Starting Interactive Session
```bash
# Once allocation is granted
srun --pty bash
# You should see: base aurorap@nodename
```

### GPU Setup (if needed)
```bash
srun --pty bash
module load PDCOLD/23.12
module load singularity/4.1.1-cpeGNU-23.12
```

### Environment Activation
```bash
conda activate myenv
```

### Running Scripts
```bash
python main.py
# or
bash test.sh
```

## Storage Management

### Check Available Space
```bash
# Check disk space
df -h .

# Check directory usage
du -h --max-depth=1 | sort -hr

# Check individual file sizes
ls -lh
```

## Background Processing

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

### Server to Local
```bash
# Move folders
scp -r aurorap@dardel-ftn01.pdc.kth.se:/cfs/klemming/home/a/aurorap/folderpath /Users/aurorap/Desktop/folderpath

# Move single file
scp aurorap@dardel-ftn01.pdc.kth.se:/cfs/klemming/home/a/aurorap/filepath/namefile /Users/aurorap/Desktop/filepath/namefile

# Move file using VPN
scp -i ~/.ssh/ed25519_vpn aurorap@dardel-ftn01.pdc.kth.se:/cfs/klemming/home/a/aurorap/filepath/filename /Users/aurorap/Desktop/filepath/filename
```

### Local to Server
```bash
# Move folders
scp -r /Users/aurorap/Desktop/folderpath aurorap@dardel-ftn01.pdc.kth.se:/cfs/klemming/home/a/aurorap/folderpath

# Move single file
scp /Users/aurorap/Desktop/filepath/namefile aurorap@dardel-ftn01.pdc.kth.se:/cfs/klemming/home/a/aurorap/filepath/namefile

# Move file using VPN
scp -i ~/.ssh/ed25519_vpn /Users/aurorap/Desktop/filepath/filename aurorap@dardel-ftn01.pdc.kth.se:/cfs/klemming/home/a/aurorap/filepath/filename
```

### Get Current Directory Path
```bash
pwd
```

## Screen Sessions (Other Servers)

Screen allows running code while closing terminal and laptop.

### Basic Screen Commands
```bash
# Enter server
ssh aurorap@servername

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

## Version Control (GitHub)

Always use GitHub Desktop for GUI operations.

### Terminal Git Commands
```bash
# Before modifying anything
git fetch
git rebase
```

## Python Parallel Processing

### Documentation
- Main: https://docs.python.org/3/library/multiprocessing.html
- Tutorial: https://www.kth.se/blogs/pdc/2019/02/parallel-programming-in-python-multiprocessing-part-1/
- Additional: https://superfastpython.com/multiprocessing-pool-python/
- Guide: https://medium.com/@mehta.kavisha/different-methods-of-multiprocessing-in-python-70eb4009a990

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

# Before functions
@line_profiler.profile
def your_function():
    pass
```

```bash
# In terminal
kernprof -l -v main.py
```

## Notes
- **Synchronous methods**: `map` and `starmap` guarantee correct output order but may have performance issues if workload is unbalanced
- **Asynchronous methods**: `apply_async` allows better load balancing
- **VPN limitation**: Mac VPN drops every 24 minutes
- **Memory allocation**: Be careful with large matrix operations on Dardel
