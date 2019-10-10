# practical_network_automation

* hide password from the console during input: 
```python
import getpass 
p = getpass.getpass(prompt="Enter your password: ")
```
* encoding and decoding arbitrary binary strings into text strings that can be safely sent by email, used as parts of URLs, or included as part of an HTTP POST request. The encoding algorithm is not the same as the uuencode program.
```python
import base64
###Encrypt a given set of credentials
def encryptcredential(pwd):
  rvalue=base64.b64encode(pwd.encode())
  return rvalue

###Decrypt a given set of credentials
def decryptcredential(pwd):
    rvalue=base64.b64decode(pwd)
    rvalue=rvalue.decode()
    return rvalue
```
* how to access an API
```python 
import requests
city = "london"
# this would give a sample data of the city that was used in the variable
urlx = "https://samples.openweathermap.org/data/2.5/weather?q=" + \
    city+"&appid=b6907d289e10d714a6e88b30761fae22"
# send the request to URL using GET Method
r = requests.get(url=urlx)
output = r.json()
# parse the valuable information from the return JSON
print("Raw JSON \n")
print(output)
print("\n")
# fetch and print latitude and longitude
citylongitude = output['coord']['lon']
citylatitude = output['coord']['lat']
print("Longitude: "+str(citylongitude) +
      " , "+"Latitude: "+str(citylatitude))
```
* Python is extensively used for interaction with infrastructure devices, Network Gear, and multiple vendors, but to have deep integration into and accessibility on any Windows platform, PowerShell will be the best choice. Python is extensively used in Linux environments, where PowerShell has a very limited support. 
* API access via powershell
```shell
#use the city of london as a reference
 $city="london"
 $urlx="https://samples.openweathermap.org/data/2.5/weather?q="+$city+"&appid=b6907d289e10d714a6e88b30761fae22"
# used to Invoke API using GET method
 $stuff = Invoke-RestMethod -Uri $urlx -Method Get
#write raw json
 $stuff
#write the output of latitude and longitude
 write-host ("Longitude: "+$stuff.coord.lon+" , "+"Latitude: "+$stuff.coord.lat)
```
* 2 important libs in python for networking: paramiko and netmiko
* netmiko(based on paramiko):
    * Successfully establish an SSH connection to the device
    * Simplify the execution of show commands and the retrieval of output data
    * Simplify execution of configuration commands including possibly commit actions
    * Do the above across a broad set of networking vendors and platforms

* netmiko example, ```send_command```
```python
from netmiko import ConnectHandler

device = ConnectHandler(device_type='cisco_ios', ip='192.168.255.249', username='cisco', password='cisco')
output = device.send_command("show version")
print (output)
device.disconnect()
```
* netmiko supports platforms like :
  * a10: A10SSH,
  * accedian: AccedianSSH,
  * alcatel_aos: AlcatelAosSSH,
  * alcatel_sros: AlcatelSrosSSH,
  * arista_eos: AristaSSH,
  * aruba_os: ArubaSSH
  * and many more

* it is possilble to configure interfaces with ```config_push```:
```python
from netmiko import ConnectHandler

print ("Before config push")
device = ConnectHandler(device_type='cisco_ios', ip='192.168.255.249', username='cisco', password='cisco')
output = device.send_command("show running-config interface fastEthernet 0/0")
print (output)

configcmds=["interface fastEthernet 0/0", "description my test"]
device.send_config_set(configcmds)

print ("After config push")
output = device.send_command("show running-config interface fastEthernet 0/0")
print (output)

device.disconnect()
```
### Ansible
* Ansible is an open-source software provisioning, configuration management, and application-deployment tool. It runs on many Unix-like systems, and can configure both Unix-like systems as well as Microsoft Windows. 
* ansible components: 
    * Inventory: This is a configuration file where you define the host information that needs to be accessed. The default file created at the time of installation is /etc/ansible/hosts.
    * Playbook: Playbooks are simply a set of instructions that we create for Ansible to configure, deploy, and manage the nodes declared in the inventory. 
    * Plays: Plays are defined tasks that are performed on a given set of nodes. A playbook consists of one or more plays.
    * Tasks: Tasks are specific configured actions executed in a playbook.
    * Variables: These are custom defined and can store values based upon execution of tasks.
    * Roles: Roles define the hierarchy of how the playbooks can be executed. For example, say as a primary role of a web server can have sub tasks/roles to install certain application based upon server types.
    
* ansible usage example:
```shell
ansible myrouters -m ping -f 5
```
* ansible provide info on node:
```shell
ansible localhost -m setup |more
```
* run shell command on several remote machines:
```
ansible servers -m shell -a "reboot"
```
* good resource on https://learn.getgrav.org/advanced/yaml
* example of playbook
```shell
-hosts:webservers
  vars:http_port:80
  max_clients:200
  remote_user:root
  tasks:-name:test connection
  ping:
```
