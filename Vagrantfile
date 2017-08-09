# -*- mode: ruby -*-
# vi: set ft=ruby

Vagrant.configure("2") do |config|

    # Default provider is VirtualBox!
    # If you want AWS, you need to populate and run e.g.
    #   . aws.sh; vagrant up --provider aws
    # Make sure you don't check in aws.sh (maybe make a copy with your "secret" data)
    # Before that, do
    #   vagrant plugin install vagrant-aws; vagrant plugin install vagrant-sshfs
    config.vm.box = "ubuntu/trusty64"
    config.vm.synced_folder ".", "/vagrant", :mount_options => ["dmode=777", "fmode=777"]

    config.vm.provider "virtualbox" do |vbox|
      config.ssh.forward_x11 = true

      # enable (uncomment) this for debugging output
      #vbox.gui = true

      # host-only network on which web browser serves files
      config.vm.network "private_network", ip: "192.168.56.101"

      vbox.cpus = 2
      vbox.memory = 8192
    end

    config.vm.provider "aws" do |aws, override|

      aws.tags["Name"] = "Eesen Transcriber"
      aws.ami = "ami-663a6e0c" # Ubuntu ("Trusty") Server 14.04 LTS AMI - US-East region
      aws.instance_type = "m3.xlarge"

      override.vm.synced_folder ".", "/vagrant", type: "sshfs", ssh_username: ENV['USER'], ssh_port: "22", prompt_for_password: "true"

      override.vm.box = "http://speechkitchen.org/dummy.box"

      # it is assumed these environment variables were set by ". aws.sh"
      aws.access_key_id = ENV['AWS_KEY']
      aws.secret_access_key = ENV['AWS_SECRETKEY']
      aws.keypair_name = ENV['AWS_KEYPAIR']
      override.ssh.username = "ubuntu"
      override.ssh.private_key_path = ENV['AWS_PEM']

      aws.terminate_on_shutdown = "true"
      aws.region = ENV['AWS_REGION']

      # https://console.aws.amazon.com/ec2/v2/home?region=us-east-1#SecurityGroups
      # Edit the security group on AWS Console; Inbound tab, add the HTTP rule
      aws.security_groups = "launch-wizard-1"

      #aws.subnet_id = "vpc-666c9a02"
      aws.region_config "us-east-1" do |region|
        #region.spot_instance = true
        region.spot_max_price = "0.1"
      end

      # this works around the error from AWS AMI vm on 'vagrant up':
      #   No host IP was given to the Vagrant core NFS helper. This is
      #   an internal error that should be reported as a bug.
      #override.nfs.functional = false
    end

    config.vm.provider "azure" do |azure, override|
      # each of the below values will default to use the env vars named as below if not specified explicitly
      azure.tenant_id = ENV['AZURE_TENANT_ID']
      azure.client_id = ENV['AZURE_CLIENT_ID']
      azure.client_secret = ENV['AZURE_CLIENT_SECRET']
      azure.subscription_id = ENV['AZURE_SUBSCRIPTION_ID']

      # For now, this brings up Ubuntu 16.04 LTS
      override.ssh.private_key_path = '~/.ssh/id_rsa'
      override.vm.box = "azure"
      #azure.vm_name = "Eesen Transcriber"
      #azure.resource_group_name = "jsalt"
      azure.location = "westus2"
      #override.vm.box = "https://github.com/azure/vagrant-azure/raw/v2.0/dummy.box"
    end

  config.vm.provision "shell", inline: <<-SHELL
    sudo apt-get update -y
    sudo apt-get upgrade -y

    if grep --quiet vagrant /etc/passwd
    then
      user="vagrant"
    else
      user="ubuntu"
    fi

    sudo apt-get install -y git make automake libtool autoconf patch subversion fuse \
       libatlas-base-dev libatlas-dev liblapack-dev sox libav-tools g++ \
       zlib1g-dev libsox-fmt-all apache2 sshfs
    sudo apt-get install -y openjdk-6-jre || sudo apt-get install -y icedtea-netx-common icedtea-netx
#    sudo apt-get install -y libtool-bin

    # If you wish to train EESEN with a GPU machine, uncomment this section to install CUDA
    # also uncomment the line that mentions cudatk-dir in the EESEN install section below
    #cd /home/${user}
    #wget -nv http://speechkitchen.org/vms/Data/cuda-repo-ubuntu1404-7-5-local_7.5-18_amd64.deb
    #dpkg -i cuda-repo-ubuntu1404-7-5-local_7.5-18_amd64.deb
    #rm cuda-repo-ubuntu1404-7-5-local_7.5-18_amd64.deb
    #apt-get update                                                                  
    #apt-get remove --purge xserver-xorg-video-nouveau                           
    #apt-get install -y cuda

    # Kaldi and others want bash - otherwise the build process fails
    [ $(readlink /bin/sh) == "dash" ] && sudo ln -s -f bash /bin/sh

    # Install Anaconda and Theano
#    wget https://3230d63b5fc54e62148e-c95ac804525aac4b6dba79b00b39d1d3.ssl.cf1.rackcdn.com/Anaconda-2.3.0-Linux-x86_64.sh
#    bash Anaconda-2.3.0-Linux-x86_64.sh -b # batch install into /home/vagrant/anaconda
    cd /home/${user}
    sudo -S -u vagrant -i /bin/bash -l -c 'bash /vagrant/Anaconda-2.3.0-Linux-x86_64.sh -b'

    # Install OpenSMILE
    cd /home/${user}
    wget -q http://audeering.com/download/1131/ -O OpenSMILE-2.1.tar.gz
    tar zxvf OpenSMILE-2.1.tar.gz

    # Get OpenSAT and GitHub repos (~/G/{coconut/pdnn})
    cd /home/${user}
    #tar zxvf /vagrant/OpenSAT.tgz
    tar zxvf /vagrant/OpenSAT2.tgz

    # get theanorc!
    cp /vagrant/.theanorc /home/${user}/

    export PATH=/home/${user}/anaconda/bin:$PATH

    # install theano
    sudo -i -u ${user} /home/${user}/anaconda/bin/conda install -y theano=0.8.2

    # assume 'conda' is installed now (get path)
    sudo -i -u ${user} /home/${user}/anaconda/bin/conda install numpy scipy mkl dill

    # get eesen-offline-transcriber
    mkdir -p /home/${user}/tools
    cd /home/${user}/tools
    git clone https://github.com/srvk/srvk-eesen-offline-transcriber
    mv srvk-eesen-offline-transcriber eesen-offline-transcriber
    # make links to EESEN
    cd eesen-offline-transcriber
    ln -s /home/${user}/eesen/asr_egs/tedlium/v2-30ms/steps .
    ln -s /home/${user}/eesen/asr_egs/tedlium/v2-30ms/utils .

    # Results (and intermediate files) are placed on the shared host folder
    mkdir -p /vagrant/{build,log,transcribe_me,src-audio}

    ln -s /vagrant/build /home/${user}/tools/eesen-offline-transcriber/build
    ln -s /vagrant/src-audio /home/${user}/tools/eesen-offline-transcriber/src-audio

    # get XFCE, xterm if we want guest VM to open windows /menus on host
    #sudo apt-get install -y xfce4-panel xterm

    # shorten paths used by vagrant ssh -c <command> commands
    # by symlinking ~/bin to here
    ln -s /home/${user}/tools/eesen-offline-transcriber /home/${user}/bin

    # get SLURM stuff
    apt-get install -y --no-install-recommends slurm-llnl < /usr/bin/yes
    /usr/sbin/create-munge-key -f
    mkdir -p /var/run/munge /var/run/slurm-llnl
    chown munge:root /var/run/munge
    chown slurm:slurm /var/run/slurm-llnl
    echo 'OPTIONS="--syslog"' >> /etc/default/munge
    cp /vagrant/conf/slurm.conf /etc/slurm-llnl/slurm.conf
    cp /vagrant/conf/reconf-slurm.sh /root/
    # 
    # Supervisor stuff needed by slurm
    # copy config first so it gets picked up
    cp /vagrant/conf/supervisor.conf /etc/supervisor.conf
    mkdir -p /etc/supervisor/conf.d
    cp /vagrant/conf/slurm.sv.conf /etc/supervisor/conf.d/
    # Now start service
    apt-get install -y supervisor
    
    # Turn off release upgrade messages
    sed -i s/Prompt=lts/Prompt=never/ /etc/update-manager/release-upgrades
    rm -f /var/lib/ubuntu-release-upgrader/*
    /usr/lib/ubuntu-release-upgrader/release-upgrade-motd
    
    # Silence error message from missing file
    touch /home/${user}/.Xauthority 

    # Provisioning runs as root; we want files to belong to '${user}'
    chown -R ${user}:${user} /home/${user}

    # Handy info
    echo ""
    echo "------------------------------------------------------------"
    echo ""
    echo "  Watching folder [...]/eesen-transcriber/transcribe_me/"
    echo "    for new files to transcribe. Output files"
    echo "    will appear alongside the original audio files"
    echo "    logs are in [...]/eesen-transcriber/log/"
    echo ""
    echo "------------------------------------------------------------"
  SHELL
end

# always monitor watched folder
Vagrant.configure("2") do |config|
  config.vm.provision "shell", run: "always", inline: <<-SHELL
    if grep --quiet vagrant /etc/passwd
    then
      user="vagrant"
    else
      user="ubuntu"
    fi

    rm -rf /var/run/motd.dynamic

    su ${user} -c "cd /home/${user}/tools/eesen-offline-transcriber && ./watch.sh >& /vagrant/log/watched.log &"
SHELL
end