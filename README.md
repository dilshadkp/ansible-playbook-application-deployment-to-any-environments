# Single Ansible Playbook to deploy application to any environments

## Description

This is an Ansible Playbook to deploy your website from GitHub to both staging and production environments in AWS.

![](https://i.ibb.co/stN9K4b/deployment.png)

## Features

- Fully Automated Playbook
- You can deploy your code to any number of EC2 instances in a single run
- This Playbook will deploy the code from GitHub to each servers in selected project environment in one-by-one manner.
- The selection of servers is based on AWS tags. To implement this Playbook in an AWS infrastructure, each server in that infra should have below two tags:
  1. ***project*** :Indicating the project in which this instance belongs to.
      - Example: project: uber,  project:ola
  2. ***env*** :Indicating the environment of the project.
      - Example: env:prod, env:stg
- You will be prompted to Enter below details:
  - Datacenter access details such as
    -  *Access Key* for AWS access
    -  *Secret Key* for AWS access
    -  *AWS Region*

  - Project details such as
    - *Project name*
    - *Project Environment(stg/prod)*

  - Server access details such as
    - *SSH user of servers*
    - *SSH port of servers*
    - *SSH key name*
  - *Git URL* in which your your project is uploaded.

## Prerequisites

- Install Ansible in Ansible Master server. [Click here for Ansible Installation steps](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html).

 ## Ansible Modules used in this Playbook
- [ec2_instance_info](https://docs.ansible.com/ansible/latest/collections/community/aws/ec2_instance_info_module.html)
- [add_host](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/add_host_module.html)
- [git](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/git_module.html)
- [copy](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/copy_module.html)
- [service](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/service_module.html)
- [debug](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/debug_module.html)

 ## Tasks defined in the Playbook

 #### Task 1
>This task will connects to the AWS with provided access key and secret key and fetch the details of all the servers which are satisfying below conditions and store them to a variable ***server_info***
> - Region is the provided region
> - *'env'* tag of server is the environment provided by you
> - *'project'* tag of the server is the project provided by you 
 
```python
     - name: "Fetch details Servers"  
      ec2_instance_info:
        aws_access_key: "{{accesskey}}"
        aws_secret_key: "{{secretkey}}"
        region: "{{region}}"
        filters:
          "tag:env": "{{env}}"
          "tag:project": "{{project}}"
      register: server_info
```
      
#### Task 2
>This task will create an in-memory inventory file with the details fetched from above task and ssh user, ssh port and ssh key name provided by you
     
```python
    - name: "Create in-memory inventory"
      add_host:
        groups: "server"
        hostname: "{{ item.public_ip_address }}"
        ansible_host: "{{ item.public_ip_address }}"
        ansible_user: "{{ssh_user}}"
        ansible_port: "{{ssh_port}}"
        ansible_private_key_file: "{{key_name}}.pem"
        ansible_ssh_common_args: "-o StrictHostKeyChecking=no"
      with_items:
        - "{{server_info.instances}}"
```

  #### Task 3
  
  > This task will clone the code from the Github URL provided by you and copy it to a directory in the server(here, I set that to /var/www/website/ in variables.vars)

```python
    - name: "clone repository from Github"
      git:
        repo: "{{git_url}}"
        dest: "{{clone_dir}}"
      register: git_info
  ```
  #### Task 4
  
>This task will stop HTTPD service in the server and wait 10 seconds to off-load that instance from the Loadbalancer
>This task will run only if there is a change between the code in Github and code in the server
> - If you want, you can modify connection draning time as per the properties of your project/Loadbalancer from variables.vars
  
```python
    - name: "off-load instance from loadbalancer"
      when: git_info.changed == true
      service:
        name: httpd
        state: stopped

    - name: "wait for connection draining"
      when: git_info.changed == true
      pause:
        seconds: "{{offload_wait}}"
```
  #### Task 5
  
>These set of tasks will perform the below taks only if there is a change between the code in Github and code in the server.
>   - Copy contents from /var/www/websites/ to document root(it is now set to /var/www/html in variables.vars)
>   - Set *owner* and *group* of files properly under docuement root
>   - Start HTTPD service
>   - Wait for 20 seconds to load that server back to the Loadbalancer. You can change this wait time as per your project requirements in variables.vars

```python
    - name: "copy contents to document root"
      when: git_info.changed == true
      copy:
        src: "{{clone_dir}}"
        dest: "{{doc_root}}"
        owner: "{{owner}}"
        group: "{{group}}"
        remote_src: true 

    - name: "Load instance back to loadbalancer"
      when: git_info.changed == true
      service:
        name: httpd
        state: restarted
        enabled: true

    - name: "wait for health check pass"
      when: git_info.changed == true
      pause:
        seconds: {{"onload_wait"}}

```
  #### Task 6

>These tasks are to print the outcome of the deployment process in the screen.

```python
    - name: "deployment status"
      when: git_info.changed == false
      debug:
        msg: " {{ansible_hostname}} {{ansible_host}} already have the latest version"

    - name: "deployment status 2"
      when: git_info.changed == true
      debug:
        msg: " {{ansible_hostname}} {{ansible_host}} DEPLOYMENT SUCCESSFULL"
```
 ## Execution
 - Put the Playbook and variables.vars in the Ansible Master server working directory.


 - Run a syntax check

```bash
ansible-playbook main.yml --syntax-check
```
 - Execute the Playbook

```bash
ansible-playbook main.yml
```

#### Sample screenshots

![](https://i.ibb.co/zfYmmqz/1.png)
![](https://i.ibb.co/YRRwd5P/2.png)

x------------------x---------------------x---------------------x-------------------x--------------------x---------------------x-------------------x

### ⚙️ Connect with Me 

<p align="center">
<a href="mailto:dilshad.lalu@gmail.com"><img src="https://img.shields.io/badge/Gmail-D14836?style=for-the-badge&logo=gmail&logoColor=white"/></a>
<a href="https://www.linkedin.com/in/dilshadkp/"><img src="https://img.shields.io/badge/LinkedIn-0077B5?style=for-the-badge&logo=linkedin&logoColor=white"/></a> 
<a href="https://www.instagram.com/dilshad_a.k.a_lalu/"><img src="https://img.shields.io/badge/Instagram-E4405F?style=for-the-badge&logo=instagram&logoColor=white"/></a>
<a href="https://wa.me/%2B919567344212?text=This%20message%20from%20GitHub."><img src="https://img.shields.io/badge/WhatsApp-25D366?style=for-the-badge&logo=whatsapp&logoColor=white"/></a><br />
