# Run in Background 

## Using nohup
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
    file_path="name_${k}_.npy"
    nohup python3 main.py $k $t >nohup_main.out &
done
```

## Using Screen Sessions (not working on Dardel)

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

