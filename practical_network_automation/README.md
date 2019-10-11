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
* good resource on yaml https://learn.getgrav.org/advanced/yaml
* example of playbook
```shell
-hosts:webservers
  vars:http_port:80
  max_clients:200
  remote_user:root
  tasks:-name:test connection
  ping:
```
    * hosts: This lists the group or managed nodes (in this case, webservers), or individual nodes separated by a space.
    * vars: This is the declaration section where we can define variables, in a similar fashion to how we define them in any other programming language. In this case, http_port: 80 means the value of 80 is assigned to the http_port variable.
    * tasks: This is the actual declaration section on what task needs to be performed on the group (or managed nodes) that was defined under the - hosts section.
    * name: This denotes the remark line used to identify a particular task.

### Network templates
* Step 1. Identifying SKU(stock keeping unit). S - small (0-100), M - medium (100-500), L - large(500+)
* Step 2 – identifying the right configuration based upon the SKU. For example, for a very small SKU, the device does not need routing configuration, or for a large SKU, we need to enable BGP routing in the device.
* Step 3 – identifying the role of the device. Let it be Loopback 99 that is an internet router.
* For a basic template, we will use Jinja2. This is a template language extensively used in Python and Ansible and is easy to understand.
```python
interface Loopback 99
description "This is switch mgmt for device {{ inventory_hostname }}"
```
```python
-sh-4.2$ more checktemplate.yml
- name: generate configs
  hosts: all
  gather_facts: false
   tasks:
     - name: Ansible config generation
       template: 
           src: vsmalltemplate.j2
           dest: "{{ inventory_hostname }}.txt"
```
* integration with python
```python
-sh-4.2$ more checkpython.py
#import libraries
import json
import sys
from collections import namedtuple
from ansible.parsing.dataloader import DataLoader
from ansible.vars.manager import VariableManager
from ansible.inventory.manager import InventoryManager
from ansible.playbook.play import Play
from ansible.executor.playbook_executor import PlaybookExecutor

def ansible_part():
    playbook_path = "checktemplate.yml"
    inventory_path = "hosts"

    Options = namedtuple('Options', ['connection', 'module_path', 'forks', 'become', 'become_method', 'become_user', 'check', 'diff', 'listhosts', 'listtasks', 'listtags', 'syntax'])
    loader = DataLoader()
    options = Options(connection='local', module_path='', forks=100, become=None, become_method=None, become_user=None, check=False,
                    diff=False, listhosts=False, listtasks=False, listtags=False, syntax=False)
    passwords = dict(vault_pass='secret')

    inventory = InventoryManager(loader=loader, sources=['inventory'])
    variable_manager = VariableManager(loader=loader, inventory=inventory)
    executor = PlaybookExecutor( 
                playbooks=[playbook_path], inventory=inventory, variable_manager=variable_manager, loader=loader, 
                options=options, passwords=passwords) 
    results = executor.run() 
    print results

def main():
    ansible_part()

sys.exit(main())
```
# Chef
* Chef is another configuration management tool that is used to automate the configuration and deployment of the infrastructure through code, as well as manage it
* Chef is client/server-based, which means the configuration can be managed on the server and clients can perform actions through pulling tasks from the server. Chef coding is done in a Ruby Domain Specific Language (DSL), which is an industry standard coding language. Ruby as a language is used to create different DSLs. For example, Ruby on Rails is considered the DSL for creating web-based applications. 

* The key components in Chef are as follows:
      *     Cookbook: This is similar to an Ansible role, and is written in Ruby to perform specific actions in the infrastructure (defined as creating a scenario for a specific infrastructure). As a role in Ansible, this defines the complete hierarchy and all the configuration tasks that need to be performed for each component in the hierarchy.
     ```python
     chef generate cookbook testcookbook
     ```
          * Attributes: These are the predefined system variables that contain values defined in the default.rb attributes file, located within the recipes folder in the specific cookbook (in this case, the location will be chef-repo/cookbooks/testcookbook/recipe/default.rb). These attributes can be overwritten in the cookbook itself and preference is given to cookbook attributes over the default values. 
          * Files: These are all the files under the [Cookbook]/files folder, and can be locally transferred to hosts running chef-client. Specific files can be transferred based upon host, platform version, and other client-specific attributes. Any files under [Cookbook]/files/default are available to all hosts.
          * Library: These are modules available in Chef for specific usage. Certain libraries can be directly invoked as they are available as inbuilt modules, whereas others need to be explicitly downloaded and installed based upon the requirements. These are available under the /libraries folder under the cookbook.
          * Metadata: Metadata defines the specifics that the Chef client and Chef server use to deploy the cookbooks on each host. This is defined in the main folder of the cookbook with the name metadata.rb.
          * Recipe: These are similar to tasks in Ansible, and are written in Ruby to perform specific actions and triggers. A recipe can be invoked from another recipe or can perform its own independent set of actions in a cookbook. A recipe must be added to a run-list to be used by chef-client and is executed in the order defined in the run-list.
          * Resources: These are predefined sets of steps for a specific purpose. These cover most of the common actions for common platforms, and additional resources can be built. One or more  resources are grouped into recipes to make a function configuration.
          * Tests: These are the unit and integration testing tools available to ensure the recipes in a cookbook are validated and perform the correct set of tasks that they were configured for. They also perform syntax validations and validate the flow of recipes in the cookbook. Some popular tools to validate Chef recipes are Test Kitchen and ChefSpec.
      * Nodes: These are components that are managed as an inventory in Chef. Nodes can consist of any component, such as servers, network devices, cloud, or virtual machines. 
