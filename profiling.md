# Python Code Profiling

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
