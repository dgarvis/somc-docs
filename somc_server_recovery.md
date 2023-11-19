
# Table of Contents

1.  [Summary](#org180c02d)
2.  [Setting Up the Linux Server](#org513a7e7)
3.  [Setup Custom Docker Images](#org1d4b595)
4.  [Restore Data](#org5e944e8)
    1.  [Restore the General System](#orgbcd9b76)
    2.  [Restore the minecraft world data from the latest incremental backup.](#orgf4b2e7c)
5.  [Starting the Minecraft Server](#org1492e38)
6.  [Network Configuration.](#org0df0961)



<a id="org180c02d"></a>

# Summary

This document will go over how to setup a Linux Server and configure it for running
SoMC Minecraft and then how to restore the data from backups.


<a id="org513a7e7"></a>

# Setting Up the Linux Server

1.  Download and Install [Ubuntu](https://ubuntu.com/) Server LTS.
2.  Ensure that the system is up-to-date.
    
        sudo apt-get upgrade
        sudo apt-get update
3.  Install Software and Tools
    
        sudo apt-get install emacs htop ranger iotop vnstat \
             ca-certificates curl gnupg git
        sudo systemctl enable vnstat
4.  Setup a User Account.
    
    I use my name for my account.
    
        useradd -m -s /bin/zsh dylan
        passwd dylan
    
    Add the user to the sudo group
    
        usermod -aG sudo dylan
5.  Change to use the user from this point on
    
        su dylan
6.  (Optional) Allow sudo without password
    
        sudo chmod +x /etc/sudoers
        emacs /etc/sudoers
        
        # uncomment the line that looks like
        # # %sudo   ALL=(ALL:ALL) NOPASSWD: ALL
        
        sudo chmod -w /etc/sudoers
7.  Configure SSH
    1.  Add you public key
        
            emacs ~/.ssh/authorized_keys
    2.  Edit SSHD to only accept public keys.
        
            sudo emacs /etc/ssh/sshd_config
            
            # enable PasswordAuthentication no
            # should set PermitRootLogin no
            
            sudo systemctl restart ssh
8.  Setup Docker
    
        sudo install-m0755 -d/etc/apt/keyrings
        curl -fsSLhttps://download.docker.com/linux/ubuntu/gpg \
             | sudo gpg --dearmor-o/etc/apt/keyrings/docker.gpg
        sudo chmod a+r /etc/apt/keyrings/docker.gpg
        
        echo\"deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
          "$(./etc/os-release &&echo"$VERSION_CODENAME")" stable"| \sudo tee/etc/apt/sources.list.d/docker.list >/dev/null
        
        sudo apt-get update
        sudo apt-get install docker-ce docker-ce-cli containerd.io \
             docker-buildx-plugin docker-compose-plugin docker-compose
    
    Then also allow the non-root user to user docker and test.
    
        sudo usermod -aG docker dylan
        docker run hello-world


<a id="org1d4b595"></a>

# Setup Custom Docker Images

Our Minecraft server has a custom docker for running backups, the discord bot
to interact with the server, and a forked version of the Minecraft-exporter
image to enable support for bedrock players.

The images should be available on docker hub by login in as **dmgarvis** with an
access token.

    docker login -u dmgarvis

Otherwise, each repo can be cloned from GitHub and then the docker image built.

    cd;
    git clone git@github.com:dgarvis/somc-manager.git;
    cd somc-manager;
    docker build --no-cache -t dmgarvis/somc-manager:latest .;
    
    cd;
    git clone git@github.com:dgarvis/minecraft-exporter.git;
    cd minecraft-exporter;
    docker build --no-cache -t dmgarvis/minecraft_exporter:latest .;
    
    cd;
    git clone git@github.com:dgarvis/minecraft-backup.git;
    cd minecraft-backup;
    docker build --no-cache -t dmgarvis/minecraft-backup:latest .;


<a id="org5e944e8"></a>

# Restore Data

Two types of backups are created, both of which are stored on [rsync.net](https://www.rsync.net/).

-   **Full System:** Includes all world files and configuration files.
    This is the primary backup needed for a restore. It is stored
    as an encrypted tar.xz.
-   **World Data:** Incremental backups of the world data.
    This is stored in the **somc** [Borg](https://www.borgbackup.org/) repo.


<a id="orgbcd9b76"></a>

## Restore the General System

1.  Download the latest full system backup
2.  Decrypt and Extract the backup (key is in ProtonPass)
    \#+beign<sub>src</sub> bash
    openssl enc -aes-256-cbc -d -in ARCHIVE.tar.xz.enc -out backup.tar.xz
    tar xf backup.tar.xz
    \#+end<sub>src</sub>
3.  From backup, restore the docker containers
    
        sudo cp backup/var/lib/docker/volumes/* /var/lib/docker/volumes/
4.  From backup, restore the minecraft config files
    
        sudo cp backup/srv/minecraft /srv


<a id="orgf4b2e7c"></a>

## Restore the minecraft world data from the latest incremental backup.

1.  Set some borg enviornment values to make the next set of commands easier.
    *(this makes an assumption that you have an SSH config file setup to access
    the rsync.net server as backup)*
    
        export BORG_REPO=backup:somc
        export BORG_PASSPHRASE=...
        export BROG_REMOTE_PATH=borg1
2.  Select the latest archive of the world
    
        archive=$(borg list --last 1 | cut -d' ' -f1)
3.  In a working directory download and extract the backup archive.
    
        mkdir /tmp/restore
        cd /tmp/restore
        borg extract ::$archive
4.  Now restore the world data
    
        sudo su
        cd /srv/minecraft/somc-data/
        rm -rf *
        mv /tmp/restore .


<a id="org1492e38"></a>

# Starting the Minecraft Server

To start the minecraft server, you should just need to run the following commands.

    cd /srv/minecraft
    docker-compose up -d


<a id="org0df0961"></a>

# Network Configuration.

Make sure to update any firewalls to allow the ports for the minecraft server, http,
and https. You may also need to update the SRV records in the name server if the
Minecraft server is running on a non-standard port.

