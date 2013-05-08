# Vagrant CentOS 6.4 setup
This document describes how to package a vagrant box configured with a minimal CentOS 6.4 installation.


## Requirements
- [VirtualBox](https://www.virtualbox.org)
- [Vagrant](http://vagrantup.com)
- [CentOS 6.4](http://mirror.raystedman.net/centos/6.4/isos/x86_64/)


## Configure VirtualBox
General:

- Name: **`centos-6.4-minimal`**
- Type: **Linux**
- Version: **Red Hat (64 bit)**

System:

- Motherboard
	- Base Memory: **512 MB**
	- Boot Order: **CD/DVD-ROM, Hard Disk**
	- Extended Features: **Enable IO APIC, Hardware clock in UTC time**

Display:

- *(use default settings)*

Storage:

- Controller: SATA **`centos-6.4-minimal.vdi`**
- Virtual Size: **64.00 GB**
- Details: **Dynamically allocated**

Audio:

- **Disable audio**

Network:

- Adapter 1:
	- **Enable Network Adapter**
	- Attached to **NAT**
	- Port Forwarding
		- Name: **ssh**
		- Protocol: **TCP**
		- Host IP: **127.0.0.1**
		- Host Port: **2222**
		- Guest IP: *(leave blank)*
		- Guest Port: **22**
		
- Adapters 2..4:
	- **Disable Network Adapter**

Ports:

- Serial Ports: **Disable Serial Port**
- USB: **Disable USB Controller**

Shared Folders:

- *(use default settings)*


## Install CentOS 6.4
Connect CD/DVD Drive to `CentOS-6.4-x86_64-minimal`.

Start VM:

- Welcome to CentOS 6.4!: **[ Install or upgrade an existing system ]**
- Disc Found: **[ Skip ]**
- CentOS: **[ OK ]**
- Language Selection: **English [ OK ]**
- Keyboard Selection: **us [ OK ]**
- Warning (Error processing drive): **[ Re-initialize all ]**
- Time Zone Selection: **[ OK ]** *(after adjusting settings)*

        [*] System clock uses UTC

        America/Chicago

- Root Password: **[ OK ]** *(after entering password)*

        Password: vagrant
        Password (confirm): vagrant

- Weak Password: **[ Use Anyway ]**
- Partitioning Type: **[ OK ]** *(after adjusting settings)*

        Use entire drive

        [*] sda

- Writing storage configuration to disk: **[ Write changes to disk ]**
- *(minimal package installation occurs)*

Take snapshot (⌘T): **Installed CentOS 6.4**

- Complete: **[ Reboot ]**


## Create/configure vagrant user
Login as root.

Enable networking:

    # ifup eth0

Create `vagrant` user:

  	# useradd vagrant
  	# passwd vagrant (vagrant)

Configure sudo via `visudo`:

  	# visudo

- Disable `requiretty`:

        # Defaults    requiretty

- Add `PATH`, `SSH_AUTH_SOCK` to `env_keep`:

        Defaults    env_keep += "PATH SSH_AUTH_SOCK"

- Enable group `wheel`:

        %wheel  ALL=(ALL)       ALL

- Enable user `vagrant`:

        vagrant ALL=(ALL)       NOPASSWD: ALL

- Write file and exit

        <ESC>
        :w
        :q

Test login via ssh from separate terminal (may have to temporarily adjust port forwarding to use port other than 2222 if vagrant box is already in use).

    $ ssh -p 2222 vagrant@127.0.0.1

Setup networking:

    $ cat /etc/sysconfig/network-scripts/ifcfg-eth0 && \
    sudo sed -i -r -e 's/^(ONBOOT=("?))no/\1yes/' /etc/sysconfig/network-scripts/ifcfg-eth0 && \
    cat /etc/sysconfig/network-scripts/ifcfg-eth0
    
    $ cat /etc/sysconfig/network && \
    sudo sed -i -r -e 's/^(HOSTNAME=)localhost\.localdomain/\1vagrant-centos-64/' /etc/sysconfig/network && \
    cat /etc/sysconfig/network
    
    $ sudo hostname vagrant-centos-64

Take snapshot (⌘T): **Created/configured vagrant user**


## Install VirtualBox Guest Additions
Devices > Install Guest Additions… (⌘D).

Install packages required for guest additions:

    $ sudo yum -y update # note: this step is time consuming
    $ sudo reboot # activate new kernel if kernel is updated
    $ sudo yum -y install man nano wget which bzip2 gcc make kernel-devel-`uname -r` autoconf

Mount CDROM & run installation script (ignore error related to Window System drivers):

    $ sudo sh -c "mount /dev/cdrom /media
    sh /media/VBoxLinuxAdditions.run"

Verify presence of kernel module:

    $ lsmod | grep vboxsf
    
    vboxsf                 37454  1 
    vboxguest             243989  2 vboxsf

Take snapshot (⌘T): **Installed VirtualBox Guest Additions**


## Configure SSH
Configure `~/.ssh`:

    $ mkdir -m 0700 ~/.ssh && cd ~/.ssh && \
    curl -k https://raw.github.com/mitchellh/vagrant/master/keys/vagrant.pub > authorized_keys && \
    chmod 0644 authorized_keys

Configure `/etc/ssh/sshd_config`:

    $ FILE=/etc/ssh/sshd_config && \
    sudo sh -c "cat \"$FILE\"
    sed -i -r -e 's/^#(UseDNS\s)yes/\1no/' \"$FILE\"
    cat \"$FILE\""

Restart `sshd` daemon:

    $ sudo /sbin/service sshd restart

Take snapshot (⌘T): **Configured SSH**


## Install Ruby
Install required packages:

    $ sudo yum -y install openssl-devel readline-devel zlib-devel

Create `~/src` directory:

    $ mkdir -p ~/src

Install `libyaml` from source:

    $ YAML=yaml-0.1.4 && \
    cd ~/src && \
    wget "http://pyyaml.org/download/libyaml/$YAML.tar.gz" && \
    tar zxvf "$YAML.tar.gz" && \
    cd "$YAML" && \
    ./configure --prefix=/usr/local && \
    make && sudo make install
	
Install Ruby from source:

    $ RUBY=ruby-1.9.3-p327 && \
    cd ~/src && \
    wget "http://ftp.ruby-lang.org/pub/ruby/1.9/$RUBY.tar.gz" && \
    tar zxvf "$RUBY.tar.gz" && \
    cd "$RUBY" && \
    ./configure --prefix=/opt/vagrant_ruby --enable-shared --disable-install-doc --with-opt-dir=/usr/local && \
    make && sudo make install

Install `ruby/openssl`:

    $ cd ~/src/"$RUBY"/ext/openssl && \
    /opt/vagrant_ruby/bin/ruby extconf.rb && \
    make && sudo make install

Install `ruby/readline`:

    $ cd ~/src/"$RUBY"/ext/readline && \
    /opt/vagrant_ruby/bin/ruby extconf.rb && \
    make && sudo make install

Install `ruby/zlib`:

    $ cd ~/src/"$RUBY"/ext/zlib && \
    /opt/vagrant_ruby/bin/ruby extconf.rb && \
    make && sudo make install

Install Rubygems from source:

    $ RUBYGEMS=rubygems-1.8.24 && \
    cd ~/src && \
    wget "http://production.cf.rubygems.org/rubygems/$RUBYGEMS.tgz" && \
    tar zxvf "$RUBYGEMS.tgz" && \
    cd "$RUBYGEMS" && \
    sudo /opt/vagrant_ruby/bin/ruby setup.rb

Create `/etc/gemrc` defaults:

    $ sudo sh -c "echo 'install: --no-rdoc --no-ri
    update:  --no-rdoc --no-ri' > /etc/gemrc"

Install Chef gem:

    $ sudo /opt/vagrant_ruby/bin/gem install chef --no-rdoc --no-ri

Take snapshot (⌘T): **Installed Ruby**


## Configure Path
Add paths to `/etc/profile.d`:

    $ sudo sh -c "echo 'pathmunge /sbin' > /etc/profile.d/path_sbin.sh
    echo 'pathmunge /usr/sbin' > /etc/profile.d/path_usr_sbin.sh
    echo 'pathmunge /opt/vagrant_ruby/bin' > /etc/profile.d/path_opt_vagrant_ruby_bin.sh" && \
    source /etc/profile

Take snapshot (⌘T): **Configured path**


## Package box add
On host machine: CD to directory containing virtual machine disk and package box.

    $ vagrant package --base centos-6.4-minimal
    $ vagrant box add centos-64-x86_64-minimal ./package.box
    $ ls -alh package.box
    
    -rw-r--r--  1 doc  staff   982M May  7 21:27 package.box
    
    $ rm package.box


## Package box remove (for updates)

    $ vagrant box remove centos-64-x86_64-minimal
    (perform steps from Package box add, above)