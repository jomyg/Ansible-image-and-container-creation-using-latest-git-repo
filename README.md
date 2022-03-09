# Ansible-image-and-container-creation-using-latest-git-repo

Playbook to insatll Docker and build a simple html application image from a dockerfile and then it pushed to your Docker Hub account.

## Ansible Modules used

- yum
- pip
- service
- git
- docker_image
- docker_container
- docker_login

## How to Use

```
git clone https://github.com/Ansible-image-and-container-creation-using-latest-git-repo.git
cd Ansible-image-and-container-creation-using-latest-git-repo
ansible-playbook -i hosts main.yml
```

## Behind the code

```sh
---

- name: "Building Docker Image and container from GITHUB"
  hosts: amazon
  become: true
  vars:
    packages:
      - git
      - pip
      - docker
    repo_url: "https://github.com/jomyg/git-flask-app.git"
    repo_dir: "/var/flaskapp/"
    docker_user: "jomyg"
    docker_password: "9495671831"
    image_name: "jomyg/flaskone"

  tasks:


    - name: " We are Installing pip, docker & git"
      yum:
        name: "{{ packages }}"
        state: present

    - name: " Installing Python extension for docker communication. Please wait"
      pip:
        name: docker-py

    - name: "Adding Ec2-user to docker group for access"
      user:
        name: "ec2-user"
        groups:
          - docker
        append: true


    - name: "Restarting and enabling Docker if need"
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
        state: present


    - name: "Creating docker Image and push To your docker-hub now. Please wait"
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
        name: "jomyg/flaskone:latest"
        source: pull
        force_source: true
      register: image_status

    - name: " Creating the Container from above fecthed image"
      when: image_status.changed == true
      docker_container:
        name: helloworddflaskapp
        image: "{{ image_name }}:latest"
        recreate: yes
        pull: yes
        published_ports:
          - "80:80"
  ```
