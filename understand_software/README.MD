
# console/interpreter
* swupdate stdout and stderr
```
app 3>&1 1>&2 2>&3//i.e. 3=1 1=2 2=3
```
* grep command
```
strace -f bash -c -- "app > xyz 2>xyz" 2>&1 | grep -A 3 'open("xyz'
```
* print environment vars
```
env
```
* set env var
```
export f=abc
```
```
export f
f=abc
```
* set env var that is not visible by child processes
```
f=abc
```
* set variable for a child process
```
f=abc ./app
```
* unset
```

unset f
```
* print all env vars(what will child process get)
```
export
env
printenv
declare -xp
```
* print all global var + local
```
declare -p
```
* unix start script
```
.bashrc //it has a symlink .bash_profile
```
