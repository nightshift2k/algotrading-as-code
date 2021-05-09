# Algotrading as Code: *a practical tutorial how to automate freqtrade - a crypto-currency algorithmic trading software*

## Disclaimer

Use everything at your own risk. I am not a financial advisor, and not responsible for anything. 

## Premise

This documentation is intended to provide necessary information, how to automatically spin up instances via Terraform and configure all necessary bits & pieces via Ansible, so at the end you have a ready to run machine with Freqtrade installed. If this is all very much new to you, please follow the excellent documentations for the forementioned software:

 - [Freqtrade](https://www.freqtrade.io/en/stable/)
 - [Terraform](https://www.terraform.io/docs/index.html)
 - [Ansible](https://docs.ansible.com/)

## Requirements

The example workflow is described on an Debian/Ubuntu based system, your mileage may vary, if you intend to use CentOS, ArchLinux, etc. please check the documentation how to install Ansible & Terraform on your preferred OS-flavour. Usage from Windows or MacOS systems is absolutely possible but not covered inhere. The probably easiest way to get everything up & running when your primary OS is Windows is WSL (Windows Subsystem for Linux) which easily allows you to run a virtual Linux environment of your choice. Please check https://wiki.ubuntu.com/WSL for more information, how to get an Ubuntu WSL instance running.

### Terraform

Terraform must be installed on the system from which the project will be executed. Start by getting Terraform from their [download page](https://www.terraform.io/downloads.html) and select the appropriate package for your system, for example:

    wget https://releases.hashicorp.com/terraform/0.15.3/terraform_0.15.3_linux_amd64.zip


Extract the binaries into a suitable location such as `/usr/local/bin` and make sure it is included in your PATH environmental variable.

    sudo unzip terraform_0.15.3_linux_amd64.zip -d /usr/local/bin
    
Test if terraform is accessible by checking the version number in the terminal via the command:
    
    terraform -v
    Terraform v0.15.3

### Ansible

The installation of Ansible on various *nix based operating systems is described in [the official Ansible documentation](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html#installing-ansible-on-specific-operating-systems), please follow the guide for your operating system.

Installing Ansible on Window OS can be a bit cumbersome, either you use Cygwin (described [here](https://phoenixnap.com/kb/install-ansible-on-windows)), or probably a better option is to use WSL as mentioned above.


## Workflow

The automated deployment is broken into two parts:

 1. Build the necessary infrastructure via Terraform
 2. Install Freqtrade with dependencies via Ansible

To outline it a bit more detailed: A terraform project will build the necessary linux instances, create SSH keys & logins and create a `hosts.cfg` file, which will be the base for Ansible to kick off the software installation.

## Step by Step Guide

### 1. Terraform Project

At the moment one Terraform Project has been published to create instances and generate all necessary files for Ansible. This project runs on Vultr infrastructure. The reason for Vultr is that they offer instances in Tokyo/Japan which have very low latency to the largest crypto exchange (Binance - also hosted in Japan).  Beside currently the pricing of Vultr is more competetive than Azure or AWS. More Terraform Projects for Azure, AWS, GCP are in the making.

On your working host, where Terraform and Ansible have been installed checkout the Vultr Terraform Project via git

    git clone https://github.com/nightshift2k/terraform-ansible-ubuntu-vultr.git

Follow the instructions in the `README.md` to setup API access to Vultr, then rename `terraform.tfvars.example` to `terraform.tfvars` and adjust all variables.

Afterwards perform

    terraform init
    terraform plan
    terraform apply

If everything went smooth proceed to the Ansible part.


### 2. Ansible

Depending which installation option you like, you can choose between a native or a docker containerized version of Freqtrade. Docker can be great if you want to run multiple containers with different versions but be a bit cumbersome for development. A parallel usage is also possible.

#### 2.1 Ansible Requirements

To create a playbook to install Freqtrade you will need the appropriate roles directly from GitHub or Ansible Galaxy. Furthermore also a community collection for Docker operations will be needed.

Install all prerequisites with `ansible-galaxy`:

    ansible-galaxy collection install community.docker
    ansible-galaxy install geerlingguy.docker
    ansible-galaxy install nightshift2k.freqtrade
    ansible-galaxy install nightshift2k.freqtrade_docker

#### 2.2 Create an Ansible Playbook for native Freqtrade installation

Please check the [`README.md`](https://github.com/nightshift2k/ansible-role-freqtrade/blob/master/README.md) of the Ansible Role `nightshift2.freqtrade` how to create and configure a playbook using this role.

In short, create a file called `freqtrade-playbook.yml` with the following content - please be aware, this is an example, your requirements may vary:

    - hosts: instances
      roles:
        - role: nightshift2k.freqtrade
        - vars:
            freqtrade_installation_directory: "/opt/freqtrade"
            freqtrade_branch: "develop"
            freqtrade_install_dev_deps: false
            freqtrade_install_plot_deps: true
            freqtrade_install_hyperopt_deps: true
            freqtrade_install_additional_pips:
              - finta
          become: yes
    
#### 2.3 Create an Ansible Playbook for a dockerized Freqtrade installation.

This is  very much similar to the above, but instead of installing freqtrade natively docker containers will be pulled from the registry or build - depending on your configuration.

Please check the [`README.md`](https://github.com/nightshift2k/ansible-role-freqtrade-docker/blob/master/README.md) of the Ansible Role `nightshift2.freqtrade_docker` how to create and configure a playbook using this role.

In short, create a file called `freqtrade-playbook.yml` with the following content - please be aware, this is an example, your requirements may vary:

    - hosts: instances
      roles:
        - role: nightshift2k.freqtrade_docker
      vars:
        freqtrade_branch: "develop"
        docker_build_directory: "/tmp/build/docker"
        build_freqtrade: true
        build_freqtrade_custom: true
        freqtrade_custom_prefix : "nightshift2k"
        freqtrade_custom_commands:
          - "RUN pip install --upgrade pip"
        freqtrade_custom_pips:
          - finta
          - ta
          - technical

After saving the playbook yml file you are ready to run ansible against your instance(s) by calling `ansible-playbook` and providing the `hosts.cfg` from the Terraform project.


#### 2.4 Run the Playbook on your instance(s)

This is basically tying it all together: `ansible-playbook` will now take the `hosts.cfg` file from Terraform and utilize the information about the host (IP's) and the credential (username + SSH Key) and run the playbook to install freqtrade in the chosen flavor (native or dockerized).

Please take note of the full path of your `hosts.cfg` as it will be needed for the command. So if your Terraform project has been executed in `/home/dev/terraform-project` the full filename you will need will very likely be `/home/dev/terraform-project/output/hosts.cfg`

**WARNING:** If you have to move the files inside the output folder, you will have to adjust the variable `ansible_ssh_private_key_file` as it contains an absolute path to the SSH Key!

Run the playbook with the following command (with filename from the above explanation):


    ansible-playbook -i /home/dev/terraform-project/output/hosts.cfg freqtrade-playbook.yml

Depending on your configuration and the instance size this might take from 5m to somwhere around 30m - especially if you run it 



## License

MIT / BSD