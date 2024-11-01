# Deployment

**Requirements**
- Docker
- Kubernetes


# Steps

Pada tahapan deployment, saya mencoba untuk menggunakan Kubernetes terlebih dahulu untuk branch staging. Lalu saya juga mencoba untuk menggunakan docker compose untuk branch produciton.

# Kubernetes

1. Setup K3S untuk master dan worker
   - K3S Master
    ```
    - name: "Install K3S on node master"
      shell: curl -sfL https://get.k3s.io | sh -

    - name: "Fetch K3S token on node master"
      shell: cat /var/lib/rancher/k3s/server/node-token
      register: master_token
      changed_when: false

    - name: "Get master node IP and Token"
      set_fact:
        master_ip: "{{ ansible_host }}"
        k3s_token: "{{ master_token.stdout }}"
    ```
  
  - K3S Agent Worker
    ```
    - name: "Install K3S agent on node worker"
      shell: |
        K3S_URL="https://{{ hostvars['master'].master_ip }}:6443"
        K3S_TOKEN="{{ hostvars['master'].k3s_token }}"
        curl -sfL https://get.k3s.io | K3S_URL=$K3S_URL K3S_TOKEN=$K3S_TOKEN sh -
    ```

  - Konfigurasi K3S KUBECONFIG
    ```
    - name: "Create .kube directory"
      file:
        path: "/home/{{ user_name }}/.kube"
        state: directory
        mode: "0755"
        owner: "{{ user_name }}"
        group: "{{ user_name }}"

    - name: "Copy K3S file yaml"
      command:
        cmd: "cp /etc/rancher/k3s/k3s.yaml {{ kubeconfig }}"
      args:
        creates: "{{ kubeconfig }}"

    - name: "Set permission for the kube config"
      file:
        path: "{{ kubeconfig }}"
        owner: "{{ user_name }}"
        group: "{{ user_name }}"
        mode: "0600"

    - name: "Add KUBECONFIG as user"
      lineinfile:
        path: "/home/{{ user_name }}/.bashrc"
        line: "export KUBECONFIG={{ kubeconfig }}"
        create: yes
        state: present
        owner: "{{ user_name }}"
        group: "{{ user_name }}"

    - name: "Add KUBECONFIG as root"
      lineinfile:
        path: "/root/.bashrc"
        line: "export KUBECONFIG={{ kubeconfig }}"
        create: yes
        state: present
    ```

  - Cek status node apakah sudah terinstal
  ![alt text](https://github.com/lawrence-olen/Test-Axiata/blob/master/3.%20Deployment/images/axiata8.png.png?raw=true)


1. Install Helm dan Helmfile

   ```
    # Install Helm
    curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

    # Verfikasi installasi helm
    helm version

    # Install Helmfile
    wget https://github.com/roboll/helmfile/releases/download/v0.144.0/helmfile_linux_amd64
    mv helmfile_linux_amd64 helmfile
    chmod 777 helmfile
    sudo cp helmfile /usr/local/bin/

    # Verifikasi installasi helmfile
    helmfile --version

    # Install Helm-diff
    helm plugin install https://github.com/databus23/helm-diff

    # Verifikasi installasi helm-diff
    helm diff release -h
   ```

2. Deployment aplikasi menggunakan helmfile [Disini saya masih menggunakan IP Server untuk melihat aplikasi sudah berjalan atau belum (http://35.197.153.227:30009/)]

- Deployment menggunakan perintah ``` helmfile apply ```
![alt text](https://github.com/lawrence-olen/Test-Axiata/blob/master/3.%20Deployment/images/axiata9.png?raw=true)

- Cek pada browser apakah sudah berjalan atau belum.
![alt text](https://github.com/lawrence-olen/Test-Axiata/blob/master/3.%20Deployment/images/axiata10.png?raw=true)


> [!NOTE]
> Beberapa perintah yang digunakan dalam konfigurasi helmfile.
> - helmfile apply
> - helmfile delete


# Docker Compose

- Sebelumnya docker dan docker compose sudah terinstall pada server.

- Melakukan clone repository terlebih dahulu.

- Masuk kedalam direktori ``` cd dist/docker-compose ```

- Gunakan docker compose untuk melakukan deployment aplikasi
  ```
  MYSQL_PASSWORD='<your_password>' docker compose --file docker-compose.yml up
  ```

- Lalu cek apakah sudah berjalan semua atau belum.
![alt text](https://github.com/lawrence-olen/Test-Axiata/blob/master/3.%20Deployment/images/axiata11.png?raw=true)

- Buka browser untuk memastikan apakah aplikasi sudah berjalan atau belum (https://axiata.olen.studentdumbways.my.id/home).
![alt text](https://github.com/lawrence-olen/Test-Axiata/blob/master/3.%20Deployment/images/axiata12.png?raw=true)


> [!NOTE]
> Beberapa perintah yang digunakan dalam konfigurasi docker compose.
> - docker compose up -d
> - docker compose ps -a
> - docker compose down