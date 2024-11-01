# Provisioning

**Requirements**
- Terraform
- Ansible

# Steps

Dalam persiapan dan pembuatan server, disini saya menggunakan tools Infrasturktur as a Code (IaC) yaitu terraform dan ansible. Saya menggunakan Google Cloud Provider (GCP) sebagai server yang akan dibuat nantinya.

**Terraform**

1. Setup Arsitektur pada file providers.tf

```
terraform {
  required_providers {
    google = {
      source = "hashicorp/google"
      version = "6.7.0"
    }
  }
}

provider "google" {
  # Configuration options
  project = var.gcp_project_id
  credentials = var.credentials
  region = var.region
  zone = var.zone
}
```
![alt text](https://github.com/lawrence-olen/Test-Axiata/blob/master/1.%20Provisioning/images/axiata1.png?raw=true)


2. Lalu buatlah file variable.tf yang digunakan untuk mendefinisikan variabel yang akan dipakai nantinya.

```
variable "region" {
  type = string
  description = "Region for Axiata Project"
  default = "asia-southeast1"
}

variable "zone" {
  type = string
  description = "Zone for Axiata Project"
  default = "asia-southeast1-a"
}
```
![alt text](https://github.com/lawrence-olen/Test-Axiata/blob/master/1.%20Provisioning/images/axiata2.png?raw=true)


3. Lalu buatlah file terraform.tfvars yang digunakan untuk mengatur nilai dari variabel yang sudah didefinisikan pada file variable.tf.

```
# OS Images for Server
images_OS = {
  "linux" = {
    "ubuntu" = "ubuntu-2204-lts"
    "debian" = "debian-12"
    "fedora" = "fedora-coreos-stable"
  }
}
```
![alt text](https://github.com/lawrence-olen/Test-Axiata/blob/master/1.%20Provisioning/images/axiata3.png?raw=true)


4.  Terakhir buatlah file main.tf yang digunakan untuk membuat resource yang ingin dikelola pada insfrastruktur.

```
# Create New VM Ubuntu
resource "google_compute_instance" "vm_instances_axiata" {
  for_each = var.vm_instances

  name = each.value.name
  machine_type = each.value.machine_type

  boot_disk {
    initialize_params {
      image = var.images_OS["linux"]["ubuntu"]
    }
  }

  # Attach VPC and IP Static
  network_interface {
    network = google_compute_network.vpc_network_axiata.id
    subnetwork = google_compute_subnetwork.subet_network_axiata.id

    access_config {
      // Epheral public IP
      nat_ip = google_compute_address.ip_static_axiata[each.value.ip_static].address
    }
  }

  # Attach Firewall
  tags = each.value.tags

  # Metadata for SSH keys
  metadata = {
    ssh-keys = "crocox:${file("${path.module}/ssh_key.pub")}"
  }
}
```
![alt text](https://github.com/lawrence-olen/Test-Axiata/blob/master/1.%20Provisioning/images/axiata4.png?raw=true)


**Ansible**

- Melakukan setup dan installasi Docker pada Server.

```
# tasks file for Setup
- hosts: all
  become: yes
  vars:
    user_name: crocox
  tasks:

  # Update System
  - name: "Update and Upgrade Server System"
    apt:
      update_cache: true
      upgrade: dist


  # Installation Docker
  - name: "Install Docker Dependencies"
    apt:
      name: "{{ item }}"
      state: present
    loop:
      - apt-transport-https
      - ca-certificates
      - openssh-client
      - curl
      - software-properties-common
      - htop

  - name: "Add Docker GPG Key"
    apt_key:
      url: https://download.docker.com/linux/ubuntu/gpg
      state: present

  - name: "Add Docker Repository"
    apt_repository:
      repo: "deb [arch=amd64] https://download.docker.com/linux/ubuntu {{ ansible_distribution_release }} stable"
      update_cache: true
      state: present

  - name: "Install Docker Engine and Plugin"
    apt:
      name: "{{ item }}"
      state: present
    loop:
      - docker-ce
      - docker-ce-cli
      - containerd.io
      - docker-buildx-plugin
      - docker-compose-plugin

  - name: "Enabled and Start Docker Service"
    systemd:
      name: "{{ item }}"
      enabled: true
      state: started
    loop:
      - docker.service
      - containerd.service

  - name: "Add user to group docker"
    shell: "sudo usermod -aG docker {{ user_name }}"
```

- Melakukan setup ssl encryption

```
- hosts: Jenkins_Server
  become: yes
  vars:
    user_name: crocox
  tasks:

    - name: "Install SSL Certbot using snap"
      snap:
        name: certbot
        classic: true
        state: present

    - name: "Prepare certbot symlink"
      command: ln -s /snap/bin/certbot /usr/bin/certbot
      args:
        removes: /usr/bin/certbot  # Jika sudah ada tidak akan membuat baru

    - name: "Trust cloudflare DN plugin"
      command: snap set certbot trust-plugin-with-root=ok

    - name: "Install cloudflare DNS plugin"
      snap:
        name: certbot-dns-cloudflare
        state: present

    - name: "Create directory secret"
      file:
        path: "/home/{{ user_name }}/.secret"
        state: directory
        mode: '0700'

    - name: "Create cloudflare credentials file"
      template:
        src: ../templates/cloudflare.ini.j2
        dest: "/home/{{ user_name }}/.secret/cloudflare.ini"

    - name: "Run Certbot to generate certificates"
      command: >
        certbot certonly
        --dns-cloudflare
        --dns-cloudflare-credentials /home/{{ user_name }}/.secret/cloudflare.ini
        --agree-tos
        --non-interactive
        --email lawrencenoman@gmail.com
        --expand
        --domains *.olen.studentdumbways.my.id -v
      timeout: 60
```

> [!NOTE]
> Beberapa perintah yang digunakan dalam konfigurasi terraform dan ansible.
> Terraform
> - terraform init
> - terraform validate
> - terraform plan
> - terraform apply
> - terraform destroy
>
> Ansible
> - ansible-galaxy role init your_directory
> - ansible-playbook --syntax-check your_file.yml
> - ansible-playbook your_file.yml

