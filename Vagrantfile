# -*- mode: ruby -*-
# vi: set ft=ruby :

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
      sudo DEBIAN_FRONTEND=noninteractive apt-get install -y mysql-server prometheus-mysqld-exporter

      # Modify MySQL bind-address
      sudo sed -i "s/^bind-address.*/bind-address = 10.11.12.101/" /etc/mysql/mysql.conf.d/mysqld.cnf
      echo "server-id = 1" | sudo tee -a /etc/mysql/mysql.conf.d/mysqld.cnf
      echo "log_bin = /var/log/mysql/mysql-bin.log" | sudo tee -a /etc/mysql/mysql.conf.d/mysqld.cnf
      sudo service mysql restart

      # Additional MySQL setup commands
      sudo mysql -e "ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'rootpassword';"
      sudo mysql -u root -prootpassword -e "CREATE USER 'repl'@'%' IDENTIFIED WITH mysql_native_password BY 'slavepassword';"
      sudo mysql -u root -prootpassword -e "GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';"
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
      sudo DEBIAN_FRONTEND=noninteractive apt-get install -y mysql-server prometheus-mysqld-exporter

      # Modify MySQL bind-address
      sudo sed -i "s/^bind-address.*/bind-address = 10.11.12.102/" /etc/mysql/mysql.conf.d/mysqld.cnf
      echo "server-id = 2" | sudo tee -a /etc/mysql/mysql.conf.d/mysqld.cnf
      sudo service mysql restart

      # Additional MySQL setup commands
      sudo mysql -e "ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'slavepassword';"
      mysql -u root -pslavepassword -e "CHANGE MASTER TO MASTER_HOST='10.11.12.101', MASTER_USER='repl', MASTER_PASSWORD='slavepassword';"
      mysql -u root -pslavepassword < /vagrant/masterdump.sql
      mysql -u root -pslavepassword -e "START SLAVE;"
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
      sudo DEBIAN_FRONTEND=noninteractive apt-get install -y mysql-server prometheus-mysqld-exporter

      # Modify MySQL bind-address
      sudo sed -i "s/^bind-address.*/bind-address = 10.11.12.103/" /etc/mysql/mysql.conf.d/mysqld.cnf
      echo "server-id = 3" | sudo tee -a /etc/mysql/mysql.conf.d/mysqld.cnf
      sudo service mysql restart

      # Additional MySQL setup commands
      sudo mysql -e "ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'slavepassword';"
      mysql -u root -pslavepassword -e "CHANGE MASTER TO MASTER_HOST='10.11.12.101', MASTER_USER='repl', MASTER_PASSWORD='slavepassword';"
      mysql -u root -pslavepassword < /vagrant/masterdump.sql
      mysql -u root -pslavepassword -e "START SLAVE;"
    SHELL
  end

  # Define the monitoring VM using VirtualBox provider
  config.vm.define "monitoring" do |monitoring|
    monitoring.vm.box = "ubuntu/focal64"
    monitoring.vm.network "private_network", ip: "10.11.12.104"
    monitoring.vm.hostname = "monitoring"
    monitoring.vm.provider "virtualbox" do |vb|
      vb.memory = "2048"
      vb.cpus = 2
    end
    monitoring.vm.provision "shell", inline: <<-SHELL
      sudo apt-get update
      sudo groupadd --system prometheus
      sudo useradd -s /sbin/nologin --system -g prometheus prometheus
    SHELL
  end
end
