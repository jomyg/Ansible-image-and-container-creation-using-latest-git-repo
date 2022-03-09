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
    repo_url: "https://github.com/jomyg/git-flask-app.git"                 ## The git repo url which we are using to the build the docker image
    repo_dir: "/var/flaskapp/"                                             ## The new repo will get cloned on remote location /var/flaskapp/
    docker_user: "jomyg"
    docker_password: "***********"
    image_name: "jomyg/flaskone"                                           ## Image name which you wanted to set

  tasks:


    - name: " We are Installing pip, docker & git"
      yum:
        name: "{{ packages }}"
        state: present

    - name: " Installing Python extension for docker communication. Please wait"        ## For docker python communication
      pip:
        name: docker-py

    - name: "Adding Ec2-user to docker group for access"                                ## For user "ec2-user" to access the remote meachine docker service
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

    - name: "Clonning the repo using {{ repo_url }}"                                  ## Clonning the repo to remote /var/flaskapp/
      git:
        repo: "{{repo_url}}"
        dest: "{{ repo_dir }}"
      register: git_status


    - name: "Logging into the docker-hub official"                                     ## Accessing the docker hub to push the new building images
      when: git_status.changed == true
      docker_login:
        username: "{{ docker_user }}"
        password: "{{ docker_password }}"
        state: present


    - name: "Creating docker Image and push To your docker-hub now. Please wait"       ## Image created using the repo files and pushed to docker hub
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

    - name: "Deleting Local Image From Build Server"                                    ## Deleting the unused image        
      when: git_status.changed == true
      docker_image:
        state: absent
        name: "{{ image_name }}"
        tag: "{{ item }}"
      with_items:
        - "{{ git_status.after }}"
        - latest

    - name: "Pulling the docker Image from hub "                                        ## After all image creation and push. we are pulling the latest image from hub
      docker_image:
        name: "jomyg/flaskone:latest"
        source: pull
        force_source: true
      register: image_status

    - name: " Creating the Container from above fecthed image"                          ## Creating container from the latest image which docker pulled from the hub 
      when: image_status.changed == true
      docker_container:
        name: helloworddflaskapp
        image: "{{ image_name }}:latest"
        recreate: yes
        pull: yes
        published_ports:
          - "80:80"
  ```
