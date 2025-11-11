# Python Parallel Processing

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
