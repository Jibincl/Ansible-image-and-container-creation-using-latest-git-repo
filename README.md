# Ansible-image-and-container-creation-using-latest-git-repo


## Description
On this repository, we are going to discuss on how to create a docker image and container associated from latest git repo. Iam using a build server to create the image and a test server to create the container.

### Ansible Modules used
1. yum
2. pip
3. service
4. git
5. docker_image
6. docker_container
7. docker_login

### How to Use

~~~
git clone https://github.com/Jibincl/Ansible-image-and-container-creation-using-latest-git-repo.git
cd Ansible-image-and-container-creation-using-latest-git-repo
ansible-playbook -i hosts main.yml
~~~

### Behind the code

### Ansible Hosts file

~~~
[build]

18.118.254.184 ansible_user="ec2-user" ansible_port=22 ansible_ssh_private_key_file="key.pem"

[test]

3.145.56.23 ansible_user="ec2-user" ansible_port=22 ansible_ssh_private_key_file="key.pem"

~~~

~~~
---

- name: "Building Docker Image in build server"
  hosts: build
  become: true
  vars:
    packages:
      - git
      - pip
      - docker
    repo_url: "https://github.com/Jibincl/git-flask-app.git"
    repo_dir: "/home/ec2-user/flaskapp/"
    docker_user: "jibincl"
    docker_password: "dEe5SH2ujB58ezt"
    image_name: "jibincl/flaskrep"

  tasks:
    - name: " installing packages"
      yum:
        name: "{{ packages }}"
        state: present

    - name: "Installing docker client for python"
      pip:
        name: docker-py

    - name: "adding ec2-user to docker group"
      user:
        name: "ec2-user"
        groups:
          - docker
        append: true


    - name: "Restarting/enabling Docker"
      service:
        name: docker
        state: started
        enabled: true

    - name: "Clonning the repo using {{ repo_url }}"
      git:
        repo: "{{repo_url}}"
        dest: "{{ repo_dir }}"
      register: git_status


    - name: "Logging into the docker-hub official"
      when: git_status.changed == true
      docker_login:
        username: "{{ docker_user }}"
        password: "{{ docker_password }}"



    - name: "Creating docker Image and push to hub"
      when: git_status.changed == true
      docker_image:
        source: build
        build:
          path: "{{ repo_dir }}"
          pull: yes
        name: "{{ image_name }}"
        tag: "{{ item }}"
        push: true
        force_tag: yes
        force_source: yes
      with_items:
        - "{{ git_status.after }}"
        - latest

    - name: "Logout from hub"
      when: git_status.changed == true
      docker_login:
        username: "{{ docker_user }}"
        password: "{{ docker_password }}"
        state: absent

    - name: "Removing docker image from build server"
      when: git_status.changed == true
      docker_image:

        name: "{{ image_name }}"
        tag: "{{ item }}"
        state: absent
      with_items:
        - "{{ git_status.after }}"
        - latest


- name: "creating container in test server"
  hosts: test
  become: true
  vars:
    packages:
      - pip
      - docker
    image_name: "jibincl/flaskrep"
  tasks:
    - name: " installing packages"
      yum:
        name: "{{ packages }}"
        state: present

    - name: "Installing docker client for python"
      pip:
        name: docker-py

    - name: "adding ec2-user to docker group"
      user:
        name: "ec2-user"
        groups:
          - docker
        append: true


    - name: "Restarting/enabling Docker"
      service:
        name: docker
        state: started
        enabled: true


    - name: "Pulling the docker Image from hub "
      docker_image:
        name: "{{image_name}}:latest"
        source: pull
        force_source: true
      register: image_status

    - name: " Creating the Container from pulled image"
      when: image_status.changed == true
      docker_container:
        name: flaskapp
        image: "{{ image_name }}:latest"
        recreate: yes
        pull: yes
        published_ports:
          - "80:5000"
~~~


Lets run the ansible playbook now,


~~~
ansible-playbook -i hosts main.yml

PLAY [Building Docker Image and container from git] *****************************************************************************************************************************************

TASK [Gathering Facts] **********************************************************************************************************************************************************************
[WARNING]: Platform linux on host 18.118.254.184 is using the discovered Python interpreter at /usr/bin/python, but future installation of another Python interpreter could change this. See
https://docs.ansible.com/ansible/2.9/reference_appendices/interpreter_discovery.html for more information.
ok: [18.118.254.184]

TASK [installing packages] ******************************************************************************************************************************************************************
ok: [18.118.254.184]

TASK [Installing docker client for python] **************************************************************************************************************************************************
ok: [18.118.254.184]

TASK [adding ec2-user to docker group] ******************************************************************************************************************************************************
ok: [18.118.254.184]

TASK [Restarting/enabling Docker] ***********************************************************************************************************************************************************
ok: [18.118.254.184]

TASK [Clonning the repo using https://github.com/Jibincl/devops-flask.git] ******************************************************************************************************************
changed: [18.118.254.184]

TASK [Logging into the docker-hub official] *************************************************************************************************************************************************
changed: [18.118.254.184]

TASK [Creating docker Image and push to hub] ************************************************************************************************************************************************
changed: [18.118.254.184] => (item=ad2b2bbc4d13f74b58f5d90308215cd2c61d43b1)
ok: [18.118.254.184] => (item=latest)

TASK [Logout from hub] **********************************************************************************************************************************************************************
changed: [18.118.254.184]

TASK [Removing docker image from build server] **********************************************************************************************************************************************
changed: [18.118.254.184] => (item=ad2b2bbc4d13f74b58f5d90308215cd2c61d43b1)
changed: [18.118.254.184] => (item=latest)

PLAY [creating container in build server] ***************************************************************************************************************************************************

TASK [Gathering Facts] **********************************************************************************************************************************************************************
[WARNING]: Platform linux on host 3.145.56.23 is using the discovered Python interpreter at /usr/bin/python, but future installation of another Python interpreter could change this. See
https://docs.ansible.com/ansible/2.9/reference_appendices/interpreter_discovery.html for more information.
ok: [3.145.56.23]

TASK [installing packages] ******************************************************************************************************************************************************************
ok: [3.145.56.23]

TASK [Installing docker client for python] **************************************************************************************************************************************************
ok: [3.145.56.23]

TASK [adding ec2-user to docker group] ******************************************************************************************************************************************************
ok: [3.145.56.23]

TASK [Restarting/enabling Docker] ***********************************************************************************************************************************************************
ok: [3.145.56.23]

TASK [Pulling the docker Image from hub] ****************************************************************************************************************************************************
changed: [3.145.56.23]

TASK [Creating the Container from pulled image] *********************************************************************************************************************************************
changed: [3.145.56.23]

PLAY RECAP **********************************************************************************************************************************************************************************
18.118.254.184             : ok=10   changed=5    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
3.145.56.23                : ok=7    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

~~~


After running the playbook, we could see that container is created on the client server


~~~
#docker container ls

CONTAINER ID   IMAGE                     COMMAND            CREATED         STATUS         PORTS                  NAMES
c68a5ece9750   jibincl/flaskrep:latest   "python3 app.py"   3 minutes ago   Up 3 minutes   0.0.0.0:80->5000/tcp   flaskapp
~~~


If we run the same whithout any changes on docker image repository, it will skip the creation and build

~~~
 PLAY [Building Docker Image and container from git] *****************************************************************************************************************************************

TASK [Gathering Facts] **********************************************************************************************************************************************************************
[WARNING]: Platform linux on host 18.118.254.184 is using the discovered Python interpreter at /usr/bin/python, but future installation of another Python interpreter could change this. See
https://docs.ansible.com/ansible/2.9/reference_appendices/interpreter_discovery.html for more information.
ok: [18.118.254.184]

TASK [installing packages] ******************************************************************************************************************************************************************
ok: [18.118.254.184]

TASK [Installing docker client for python] **************************************************************************************************************************************************
ok: [18.118.254.184]

TASK [adding ec2-user to docker group] ******************************************************************************************************************************************************
ok: [18.118.254.184]

TASK [Restarting/enabling Docker] ***********************************************************************************************************************************************************
ok: [18.118.254.184]

TASK [Clonning the repo using https://github.com/Jibincl/devops-flask.git] ******************************************************************************************************************
ok: [18.118.254.184]

TASK [Logging into the docker-hub official] *************************************************************************************************************************************************
skipping: [18.118.254.184]

TASK [Creating docker Image and push to hub] ************************************************************************************************************************************************
skipping: [18.118.254.184] => (item=0bcb64af45783c5098e61f0adf8b8b76205793c5)
skipping: [18.118.254.184] => (item=latest)

TASK [Logout from hub] **********************************************************************************************************************************************************************
skipping: [18.118.254.184]

TASK [Removing docker image from build server] **********************************************************************************************************************************************
skipping: [18.118.254.184] => (item=0bcb64af45783c5098e61f0adf8b8b76205793c5)
skipping: [18.118.254.184] => (item=latest)

PLAY [creating container in build server] ***************************************************************************************************************************************************

TASK [Gathering Facts] **********************************************************************************************************************************************************************
[WARNING]: Platform linux on host 3.145.56.23 is using the discovered Python interpreter at /usr/bin/python, but future installation of another Python interpreter could change this. See
https://docs.ansible.com/ansible/2.9/reference_appendices/interpreter_discovery.html for more information.
ok: [3.145.56.23]

TASK [installing packages] ******************************************************************************************************************************************************************
ok: [3.145.56.23]

TASK [Installing docker client for python] **************************************************************************************************************************************************
ok: [3.145.56.23]

TASK [adding ec2-user to docker group] ******************************************************************************************************************************************************
ok: [3.145.56.23]

TASK [Restarting/enabling Docker] ***********************************************************************************************************************************************************
ok: [3.145.56.23]

TASK [Pulling the docker Image from hub] ****************************************************************************************************************************************************
ok: [3.145.56.23]

TASK [Creating the Container from pulled image] *********************************************************************************************************************************************
skipping: [3.145.56.23]

PLAY RECAP **********************************************************************************************************************************************************************************
18.118.254.184             : ok=6    changed=0    unreachable=0    failed=0    skipped=4    rescued=0    ignored=0
3.145.56.23                : ok=6    changed=0    unreachable=0    failed=0    skipped=1    rescued=0    ignored=0

~~~


![image](https://user-images.githubusercontent.com/100774483/165181400-960f4d30-4473-4ee9-a88f-d7972e8e68ca.png)


## Conclusion

In this tutorial, we discussed about creating docker image and container associated from latest git repo. If developer do any change on the dockerfile which placed on git. This will recreate the image and container without skipping.


