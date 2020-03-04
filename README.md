# S390x Support for AWX
These are the directions for adding support for S390x to AWX ***1.0.7***. While support for other AWX versions have not been tested
you can uses these modifications as a reference.


## Modifications 
There are quite a few changes to the official AWX repo that need to be made to add support for S390x. Below you will find a detailed list
of all the changes. Some require adding files to the repo, while others modify existing files.  


#### Image Build Folder
Under the `installer/roles/image_build/` folder the following modifications need to take place:

* `Files` Folder
    * `Dockerfile.sdist`
        * Change line 1 from `FROM centos:7` to `FROM s390x/clefos:7`
        * Remove line 4, `RUN yum install -y https://centos7.iuscommunity.org/ius-release.rpm`
        * Change line 9 from `git2u-core \` to `git \`
        * Remove line 14, `RUN curl --silent --location https://rpm.nodesource.com/setup_6.x | bash -`
        * Change `CMD ["make sdist"]` to `CMD ["make -i sdist"]`
        * If behind a http proxy, add this line before the node.js install, so you can pull git@ links, `RUN git config --global url.https://github.com/.insteadOf git://github.com/`
    * `Dockerfile.rabbitmq`
        * Go to `https://github.com/ansible/awx-rabbitmq` and grab Dockerfile
        * Rename `Dockerfile` to `Dockerfile.rabbitmq`
        * Remove line 1, `ARG RABBITMQ_VERSION`
        * Change `FROM rabbitmq:${RABBITMQ_VERSION}-management-alpine` to `FROM s390x/rabbitmq:3.7.4-management-alpine`
        * Change `ADD launch.sh /launch.sh` to `ADD launch_rabbitmq.sh /launch_rabbitmq.sh`
        * Change `CMD /launch.sh` to `CMD /launch_rabbitmq.sh`
    * `launch_rabbitmq.sh`
        * Go to `https://github.com/ansible/awx-rabbitmq` and grab launch.sh
        * Rename `launch.sh` to `launch_rabbitmq.sh`
* `Templates` Folder
    * `Dockerfile.j2`
        * Change line 1 from `FROM centos:7` to `FROM s390x/clefos:7`
        * Change line 12 from `ADD https://github.com/krallin/tini/releases/download/v0.14.0/tini /tini` to `ADD https://github.com/krallin/tini/releases/download/v0.16.1/tini-s390x /tini`
        * Remove `ADD ansible.repo /etc/yum.repos.d/ansible.repo`
        * Remove `ADD RPM-GPG-KEY-ansible-release /etc/pki/rpm-gpg/RPM-GPG-KEY-ansible-release`
        * Remove `yum -y install https://centos7.iuscommunity.org/ius-release.rpm && \`
        * Change `yum -y localinstall http://download.postgresql.org/pub/repos/yum/9.6/redhat/rhel-7-x86_64/pgdg-centos96-9.6-3.noarch.rpm && \` to `yum -y install postgresql \`
        * Change this part of the line from `yum -y install ansible git2u-core` to `yum -y install ansible git`
* `Tasks` Folder
    * `main.yml`
        * Change line 80 from `shell: make sdist` to `shell: make -i sdist`
        * Under the block of code `- name: Set awx_task image name` add
            ``` 
            - name: Set rabbitmq image name
              set_fact: 
                s390x_rabbitmq_image: "{{ s390x_rabbitmq_image|default('awx_rabbitmq') }}"
            ```
        * Under the block of code `- name: Template task Dockerfile` add
            ```
            - name: Stage Dockerfile.rabbitmq
              copy:
                src: Dockerfile.rabbitmq
                dest: "{{ docker_base_path }}/Dockerfile.rabbitmq"
                mode: '0700'
              delegate_to: localhost
            ```
        * Under the block of code `- name: Stage launch_awx_task` add
            ```
            - name: Stage launch_rabbitmq
              copy:
                src: launch_rabbitmq.sh
                dest: "{{ docker_base_path }}/launch_rabbitmq.sh"
                mode: '0700'
              delegate_to: localhost
            ```
        * Under the block of code `- name: Build base task image` add
            ```
            - name: Build rabbitmq image
              docker_image:
                buildargs:
                  http_proxy: "{{ http_proxy | default('') }}"
                  https_proxy: "{{ https_proxy | default('') }}"
                  no_proxy: "{{ no_proxy | default('') }}"
                path: "{{ docker_base_path }}"
                dockerfile: Dockerfile.rabbitmq
                name: "{{ s390x_rabbitmq_image }}"
                tag: 3.7.4
                pull: no
              delegate_to: localhost
            ```
        * Under the block of code `- name: Tag task and web images as latest` add
            ```
            - name: Tag rabbitmq image as latest
              command: "docker tag {{ item }}:3.7.4 {{ item }}:latest"
              delegate_to: localhost
              with_items:
                - "{{ s390x_rabbitmq_image }}"
            ```

#### Image Push Folder
Under the `installer/roles/image_push/` folder the following modifications need to take place:

* `Tasks` Folder
    * `main.yml`
        * Under the block of code `- name: Remove task image` and before `delegate_to: localhost` add
            ```
            - name: Remove rabbitmq image
              docker_image:
                name: "{{ docker_registry }}/{{ s390x_rabbitmq_image }}"
                tag: 3.7.4
                state: absent
            ```
        * Under the block of code `- name: Tag and push task image to registry` and before `delegate_to: localhost` add
            ```
            - name: Tag and push rabbitmq image to registry
              docker_image:
                name: "{{ s390x_rabbitmq_image }}"
                repository: "{{ docker_registry }}/{{ s390x_rabbitmq_image }}"
                tag: "{{ item }}"
                push: yes
              with_items:
                - "latest"
                - 3.7.4
            ```
        * If you are using a custom registry to store docker files you need to remove all instances of `{{ docker_registry_repository }}/` from the file

#### Local Docker Folder
Under the `installer/roles/local_docker/` folder the following modifications need to take place:

* `Defaults` Folder
    * `main.yml`
        * If you are using a custom registry to pull docker files from then change `rabbitmq_image: "ansible/awx_rabbitmq:{{rabbitmq_version}}"` to `"URL_TO_REGISTRY/awx_rabbitmq:{{rabbitmq_version}}"`
* `Templates` Folder
    * `docker-compose.yml.j2`
        * Change line 132 from, `image: memcached:alpine` to `image: s390x/memcached:alpine`
        * Change line 137 from, `image: postgres:9.6` to `image: s390x/postgres:9.6`
* `Tasks` Folder
    * `standalone.yml`
        * Change line 7 from, `image: postgres:9.6` to `image: s390x/postgres:9.6`
        * Change line 34 from, `image: memcached:alpine` to `image: s390x/memcached:alpine`

#### Kubernetes Folder
Under the `installer/roles/kubernetes/` folder the following modifications need to take place:

* `Defaults` Folder
    * `main.yml`
        * If you are using a custom registry to pull docker files from then change `"ansible/awx_rabbitmq"` to `"URL_TO_REGISTRY/awx_rabbitmq"`
        * Change line 21 from, `kubernetes_memcached_image: "memcached"` to `kubernetes_memcached_image: "s390x/memcached"`
* `Templates` Folder
    * `deployment.yml.j2`
        * Right after line 252, add
            ```
            nodeSelector:
                beta.kubernetes.io/arch: s390x
            imagePullSecrets:
                - name: IMAGE_PULL_SECRET_NAME
            ```
             **NOTE:** If your pulling from a custom registry, you will need to create a kubernetes secret to allow the system to pull the images.
             Here is an example of the code that needs to be issued to add custom image pull secret to kubernetes:
             ```
             kubectl create secret docker-registry IMAGE_PULL_SECRET_NAME --docker-server=URL_TO_REGISTRY --docker-username=TEMP.TEMP@ibm.com --docker-password=TEMP_PASSWORD --docker-email=TEMP.TEMP@ibm.com
             ``` 
* `Tasks` Folder
    * `main.yml`
        * If you are using a custom registry to store docker files you need to remove all instances of `{{ docker_registry_repository }}/` from the file
        * Right below `--tiller-namespace {{ tiller_namespace }} \` add
            ```
            --set image=s390x/postgres \
            --set imageTag=9.6 \
            ```
        * Change `--set persistence.size={{ pg_volume_capacity|default('5')}}Gi \` to `--set persistence.existingClaim=CLAIM_NAME \`
          **NOTE:** You will need to create a PV and PVC so that postgres can have a persistent storage on kubernetes. Here is a example of the code that is
          needed ti add a persistent storage on kubernetes:
          ```
          kind: PersistentVolume
          apiVersion: v1
          metadata:
            name: awx-postgres-pv
          spec:
            capacity:
              storage: 50Gi
            volumeMode: Filesystem
            accessModes:
              - ReadWriteOnce
            persistentVolumeReclaimPolicy: Retain
            claimRef:
              name: awx-postgres-retain
              namespace: awx
            nfs:
              path: PATH_TO_SAVE_DATA
              server: IP_REMOTE_SERVER
          ```
          ```
          kind: PersistentVolumeClaim
          apiVersion: v1
          metadata:
            name: CLAIM_NAME
            namespace: awx
          spec:
            accessModes:
              - ReadWriteOnce
            volumeMode: Filesystem
            resources:
              requests:
                storage: 50Gi
          ```
        * Add `--tls \` after `--set persistence.existingClaim=CLAIM_NAME \`

#### Makefile
In the `root` of the repo you should find a file called `Makefile`, following modifications need to take place:

* Change line 17 from `VERSION3DOT=$(shell git describe --long --first-parent | sed 's/\-g.*//' | sed 's/\-/\./')` to `VERSION3DOT?= 1.0.7`

 
#### Inventory File
Under the `installer/` folder you will find the `inventory` file. There are some changes that are needed to add support for S390x. 
Please refer to the offical install documentation for all parameters that should be changed. The changes below are just to enable support for S390x.

* `Univerisal Install Changes`
    * At the bottom on the file on a new line add 
        ```
        task_image=awx_task
        web_image=awx_web
        awx_version=1.0.7
        ```
    * If pulling/pushing to a custom registry, change lines 52 through 54.
        * Change line 52 from `# docker_registry=172.30.1.1:5000` to `docker_registry=URL_TO_REGISTRY`
        * Leave line 53 commented, since we do not use it
        * Change line 54 from `# docker_registry_username=developer` to `docker_registry_username=USERNAME`
        
        **NOTE:** You will pass in your docker_registry_password via the command line when you run the playbook
* `Local Docker Install Changes`
    * Comment out lines 9 and 10 as they are not needed in local docker install
* `Kubernetes Install Changes`
    * Change line 9 from `dockerhub_base=ansible` to `dockerhub_base=URL_TO_REGISTRY`
    * Uncomment lines 21 to 23 and make changes based on your cluster. Please refer to official documentation for more details