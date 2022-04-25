# Ansible-image-and-container-creation-using-latest-git-repo


### Description
On this repository, we are going to discuss on how to create a docker image and container associated from latest git repo.

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

~~~
---

- name: "Building Docker Image and container from git"
  hosts: all
  become: true
  vars:
    packages:
      - git
      - pip
      - docker
    repo_url: "https://github.com/Jibincl/git-flask-app.git"           ## The git repo url which we are using to the build the docker image
    repo_dir: "/home/ec2-user/flaskapp/"                               ## The new repo will get cloned on remote location /home/ec2-user/flaskapp/
    docker_user: "jibincl"
    docker_password: "dEe5SH2ujB58ezt"
    image_name: "jibincl/flaskrep"                                     ## Image name which you wanted to set

  tasks:
    - name: " installing packages"
      yum:
        name: "{{ packages }}"
        state: present

    - name: "Installing docker client for python"                       ## For docker python communication                 
      pip:
        name: docker-py

    - name: "adding ec2-user to docker group"                           ## For user "ec2-user" to access the remote meachine docker service
      user:
        name: "ec2-user"
        groups:
          - docker
        append: true


    - name: "Restarting/enabling Docker"
      service:
        name: docker
        state: restarted
        enabled: true

    - name: "Clonning the repo using {{ repo_url }}"               ## Clonning the repo 
      git:
        repo: "{{repo_url}}"
        dest: "{{ repo_dir }}"
      register: git_status


    - name: "Logging into the docker-hub official"                 ## Accessing the docker hub to push the new building images
      docker_login:
        username: "{{ docker_user }}"
        password: "{{ docker_password }}"
        state: present


    - name: "Creating docker Image and push to hub"                 ## Image created using the repo files and pushed to docker hub
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

    - name: "Deleting Local Image From Build Server"            ## Deleting the unused image
      when: git_status.changed == true
      docker_image:
        state: absent
        name: "{{ image_name }}"
        tag: "{{ item }}"
      with_items:
        - "{{ git_status.after }}"
        - latest

    - name: "Pulling the docker Image from hub "                ## After all image creation and push. we are pulling the latest image from hub
      docker_image:
        name: "jibincl/flaskrep:latest"
        source: pull
        force_source: true
      register: image_status

    - name: " Creating the Container from pulled image"        ## Creating container from the latest image which docker pulled from the hub 
      when: image_status.changed == true
      docker_container:
        name: flaskapp
        image: "{{ image_name }}:latest"
        recreate: yes
        pull: yes
        published_ports:
          - "81:5000"
~~~


Lets run the ansible playbook now,


~~~
ansible-playbook -i hosts main.yml

PLAY [Building Docker Image and container from git] *****************************************************************************************************************************************

TASK [Gathering Facts] **********************************************************************************************************************************************************************
[WARNING]: Platform linux on host 3.145.85.53 is using the discovered Python interpreter at /usr/bin/python, but future installation of another Python interpreter could change this. See
https://docs.ansible.com/ansible/2.9/reference_appendices/interpreter_discovery.html for more information.
ok: [3.145.85.53]

TASK [installing packages] ******************************************************************************************************************************************************************
ok: [3.145.85.53]

TASK [Installing docker client for python] **************************************************************************************************************************************************
ok: [3.145.85.53]

TASK [adding ec2-user to docker group] ******************************************************************************************************************************************************
ok: [3.145.85.53]

TASK [Restarting/enabling Docker] ***********************************************************************************************************************************************************
changed: [3.145.85.53]

TASK [Clonning the repo using https://github.com/Jibincl/devops-flask.git] ******************************************************************************************************************
changed: [3.145.85.53]

TASK [Logging into the docker-hub official] *************************************************************************************************************************************************
ok: [3.145.85.53]

TASK [Creating docker Image and push to hub] ************************************************************************************************************************************************
changed: [3.145.85.53] => (item=b1b43b0ab696b2102253a5a83ad6e6664d94e036)
ok: [3.145.85.53] => (item=latest)

TASK [Deleting Local Image From Build Server] ***********************************************************************************************************************************************
changed: [3.145.85.53] => (item=b1b43b0ab696b2102253a5a83ad6e6664d94e036)
changed: [3.145.85.53] => (item=latest)

TASK [Pulling the docker Image from hub] ****************************************************************************************************************************************************
changed: [3.145.85.53]

TASK [Creating the Container from pulled image] *********************************************************************************************************************************************
changed: [3.145.85.53]

PLAY RECAP **********************************************************************************************************************************************************************************
3.145.85.53                : ok=11   changed=6    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
~~~


After running the playbook, we could see that container is created on the client server

~~~
 docker container ls
CONTAINER ID   IMAGE                     COMMAND            CREATED          STATUS          PORTS                  NAMES
382bd2d074cc   jibincl/flaskrep:latest   "python3 app.py"   32 minutes ago   Up 32 minutes   0.0.0.0:80->5000/tcp   flaskapp
~~~

If we run the same whithout any changes on docker image repository, it will skip the creation and build

~~~
 ansible-playbook -i hosts main.yml

PLAY [Building Docker Image and container from git] *****************************************************************************************************************************************

TASK [Gathering Facts] **********************************************************************************************************************************************************************
[WARNING]: Platform linux on host 3.145.85.53 is using the discovered Python interpreter at /usr/bin/python, but future installation of another Python interpreter could change this. See
https://docs.ansible.com/ansible/2.9/reference_appendices/interpreter_discovery.html for more information.
ok: [3.145.85.53]

TASK [installing packages] ******************************************************************************************************************************************************************
ok: [3.145.85.53]

TASK [Installing docker client for python] **************************************************************************************************************************************************
ok: [3.145.85.53]

TASK [adding ec2-user to docker group] ******************************************************************************************************************************************************
ok: [3.145.85.53]

TASK [Restarting/enabling Docker] ***********************************************************************************************************************************************************
changed: [3.145.85.53]

TASK [Clonning the repo using https://github.com/Jibincl/devops-flask.git] ******************************************************************************************************************
ok: [3.145.85.53]

TASK [Logging into the docker-hub official] *************************************************************************************************************************************************
ok: [3.145.85.53]

TASK [Creating docker Image and push to hub] ************************************************************************************************************************************************
skipping: [3.145.85.53] => (item=b1b43b0ab696b2102253a5a83ad6e6664d94e036)
skipping: [3.145.85.53] => (item=latest)

TASK [Deleting Local Image From Build Server] ***********************************************************************************************************************************************
skipping: [3.145.85.53] => (item=b1b43b0ab696b2102253a5a83ad6e6664d94e036)
skipping: [3.145.85.53] => (item=latest)

TASK [Pulling the docker Image from hub] ****************************************************************************************************************************************************
ok: [3.145.85.53]

TASK [Creating the Container from pulled image] *********************************************************************************************************************************************
skipping: [3.145.85.53]

PLAY RECAP **********************************************************************************************************************************************************************************
3.145.85.53                : ok=8    changed=1    unreachable=0    failed=0    skipped=3    rescued=0    ignored=0

~~~

