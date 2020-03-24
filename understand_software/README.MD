
# console/interpreter
* swupdate stdout and stderr
```
app 3>&1 1>&2 2>&3//i.e. 3=1 1=2 2=3
```
* 
```
strace -f bash -c -- "app > xyz 2>xyz" 2>&1 | grep -A 3 'open("xyz'
```
