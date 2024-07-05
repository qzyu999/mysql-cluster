# -*- mode: ruby -*-
# vi: set ft=ruby :

# # All Vagrant configuration is done below. The "2" in Vagrant.configure
# # configures the configuration version (we support older styles for
# # backwards compatibility). Please don't change it unless you know what
# # you're doing.
# Vagrant.configure("2") do |config|
#   # The most common configuration options are documented and commented below.
#   # For a complete reference, please see the online documentation at
#   # https://docs.vagrantup.com.

#   # Every Vagrant development environment requires a box. You can search for
#   # boxes at https://vagrantcloud.com/search.
#   config.vm.box = "base"

#   # Disable automatic box update checking. If you disable this, then
#   # boxes will only be checked for updates when the user runs
#   # `vagrant box outdated`. This is not recommended.
#   # config.vm.box_check_update = false

#   # Create a forwarded port mapping which allows access to a specific port
#   # within the machine from a port on the host machine. In the example below,
#   # accessing "localhost:8080" will access port 80 on the guest machine.
#   # NOTE: This will enable public access to the opened port
#   # config.vm.network "forwarded_port", guest: 80, host: 8080

#   # Create a forwarded port mapping which allows access to a specific port
#   # within the machine from a port on the host machine and only allow access
#   # via 127.0.0.1 to disable public access
#   # config.vm.network "forwarded_port", guest: 80, host: 8080, host_ip: "127.0.0.1"

#   # Create a private network, which allows host-only access to the machine
#   # using a specific IP.
#   # config.vm.network "private_network", ip: "192.168.33.10"

#   # Create a public network, which generally matched to bridged network.
#   # Bridged networks make the machine appear as another physical device on
#   # your network.
#   # config.vm.network "public_network"

#   # Share an additional folder to the guest VM. The first argument is
#   # the path on the host to the actual folder. The second argument is
#   # the path on the guest to mount the folder. And the optional third
#   # argument is a set of non-required options.
#   # config.vm.synced_folder "../data", "/vagrant_data"

#   # Disable the default share of the current code directory. Doing this
#   # provides improved isolation between the vagrant box and your host
#   # by making sure your Vagrantfile isn't accessible to the vagrant box.
#   # If you use this you may want to enable additional shared subfolders as
#   # shown above.
#   # config.vm.synced_folder ".", "/vagrant", disabled: true

#   # Provider-specific configuration so you can fine-tune various
#   # backing providers for Vagrant. These expose provider-specific options.
#   # Example for VirtualBox:
#   #
#   # config.vm.provider "virtualbox" do |vb|
#   #   # Display the VirtualBox GUI when booting the machine
#   #   vb.gui = true
#   #
#   #   # Customize the amount of memory on the VM:
#   #   vb.memory = "1024"
#   # end
#   #
#   # View the documentation for the provider you are using for more
#   # information on available options.

#   # Enable provisioning with a shell script. Additional provisioners such as
#   # Ansible, Chef, Docker, Puppet and Salt are also available. Please see the
#   # documentation for more information about their specific syntax and use.
#   # config.vm.provision "shell", inline: <<-SHELL
#   #   apt-get update
#   #   apt-get install -y apache2
#   # SHELL
#   config.vm.box = "ubuntu/focal64"
# end

Vagrant.configure("2") do |config|
  # Define the master VM
  config.vm.define "mysql-master" do |master|
    master.vm.box = "ubuntu/focal64"
    master.vm.network "private_network", ip: "10.11.12.101"
    master.vm.hostname = "mysql-master"
    master.vm.provider "virtualbox" do |vb|
      vb.memory = "1024"
      vb.cpus = 1
    end
    master.vm.provision "shell", inline: <<-SHELL
      sudo apt-get update
      sudo DEBIAN_FRONTEND=noninteractive apt-get install -y mysql-server
      sudo sed -i "s/^bind-address.*/bind-address = 10.11.12.101/" /etc/mysql/mysql.conf.d/mysqld.cnf
      echo "server-id = 1" | sudo tee -a /etc/mysql/mysql.conf.d/mysqld.cnf
      echo "log_bin = /var/log/mysql/mysql-bin.log" | sudo tee -a /etc/mysql/mysql.conf.d/mysqld.cnf
      sudo service mysql restart
      sudo mysql -e "ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'rootpassword';"
      sudo mysql -u root -prootpassword -e "CREATE USER 'repl'@'%' IDENTIFIED WITH mysql_native_password BY 'slavepassword';"
      sudo mysql -u root -prootpassword -e "GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';"
      mysqldump -u root -prootpassword --all-databases --master-data > /vagrant/masterdump.sql
      # Create .my.cnf for root user
      echo "[client]" > /home/vagrant/.my.cnf
      echo "user=root" >> /home/vagrant/.my.cnf
      echo "password=rootpassword" >> /home/vagrant/.my.cnf
      chmod 600 /home/vagrant/.my.cnf
    SHELL
  end

  # Define the first slave VM
  config.vm.define "mysql-slave1" do |slave1|
    slave1.vm.box = "ubuntu/focal64"
    slave1.vm.network "private_network", ip: "10.11.12.102"
    slave1.vm.hostname = "mysql-slave1"
    slave1.vm.provider "virtualbox" do |vb|
      vb.memory = "1024"
      vb.cpus = 1
    end
    slave1.vm.provision "shell", inline: <<-SHELL
      sudo apt-get update
      sudo DEBIAN_FRONTEND=noninteractive apt-get install -y mysql-server
      sudo sed -i "s/^bind-address.*/bind-address = 10.11.12.102/" /etc/mysql/mysql.conf.d/mysqld.cnf
      echo "server-id = 2" | sudo tee -a /etc/mysql/mysql.conf.d/mysqld.cnf
      sudo service mysql restart
      sudo mysql -e "ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'slavepassword';"
      mysql -u root -pslavepassword -e "CHANGE MASTER TO MASTER_HOST='10.11.12.101', MASTER_USER='repl', MASTER_PASSWORD='slavepassword';"
      mysql -u root -pslavepassword < /vagrant/masterdump.sql
      mysql -u root -pslavepassword -e "START SLAVE;"
      # Create .my.cnf for root user
      echo "[client]" > /home/vagrant/.my.cnf
      echo "user=root" >> /home/vagrant/.my.cnf
      echo "password=rootpassword" >> /home/vagrant/.my.cnf
      chmod 600 /home/vagrant/.my.cnf
    SHELL
  end

  # Define the second slave VM
  config.vm.define "mysql-slave2" do |slave2|
    slave2.vm.box = "ubuntu/focal64"
    slave2.vm.network "private_network", ip: "10.11.12.103"
    slave2.vm.hostname = "mysql-slave2"
    slave2.vm.provider "virtualbox" do |vb|
      vb.memory = "1024"
      vb.cpus = 1
    end
    slave2.vm.provision "shell", inline: <<-SHELL
      sudo apt-get update
      sudo DEBIAN_FRONTEND=noninteractive apt-get install -y mysql-server
      sudo sed -i "s/^bind-address.*/bind-address = 10.11.12.103/" /etc/mysql/mysql.conf.d/mysqld.cnf
      echo "server-id = 3" | sudo tee -a /etc/mysql/mysql.conf.d/mysqld.cnf
      sudo service mysql restart
      sudo mysql -e "ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'slavepassword';"
      mysql -u root -pslavepassword -e "CHANGE MASTER TO MASTER_HOST='10.11.12.101', MASTER_USER='repl', MASTER_PASSWORD='slavepassword';"
      mysql -u root -pslavepassword < /vagrant/masterdump.sql
      mysql -u root -pslavepassword -e "START SLAVE;"
      # Create .my.cnf for root user
      echo "[client]" > /home/vagrant/.my.cnf
      echo "user=root" >> /home/vagrant/.my.cnf
      echo "password=rootpassword" >> /home/vagrant/.my.cnf
      chmod 600 /home/vagrant/.my.cnf
    SHELL
  end
end
