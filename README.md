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
    repo_url: "https://github.com/Jibincl/devops-flask.git"
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
        state: restarted
        enabled: true

    - name: "Clonning the repo using {{ repo_url }}"
      git:
        repo: "{{repo_url}}"
        dest: "{{ repo_dir }}"
      register: git_status


    - name: "Logging into the docker-hub official"
      docker_login:
        username: "{{ docker_user }}"
        password: "{{ docker_password }}"
        state: present


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

    - name: "Deleting Local Image From Build Server"
      when: git_status.changed == true
      docker_image:
        state: absent
        name: "{{ image_name }}"
        tag: "{{ item }}"
      with_items:
        - "{{ git_status.after }}"
        - latest

    - name: "Pulling the docker Image from hub "
      docker_image:
        name: "jibincl/flaskrep:latest"
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
          - "81:5000"
~~~
