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
      sudo DEBIAN_FRONTEND=noninteractive apt-get install -y mysql-server

      # Modify MySQL bind-address
      sudo sed -i "s/^bind-address.*/bind-address = 10.11.12.101/" /etc/mysql/mysql.conf.d/mysqld.cnf
      echo "server-id = 1" | sudo tee -a /etc/mysql/mysql.conf.d/mysqld.cnf
      echo "log_bin = /var/log/mysql/mysql-bin.log" | sudo tee -a /etc/mysql/mysql.conf.d/mysqld.cnf
      sudo service mysql restart

      # Additional MySQL setup commands
      sudo mysql -e "ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'rootpassword';"
      sudo mysql -u root -prootpassword -e "CREATE USER 'repl'@'%' IDENTIFIED WITH mysql_native_password BY 'slavepassword';"
      sudo mysql -u root -prootpassword -e "GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';"
      sudo mysql -u root -prootpassword -e "CREATE USER 'mysqld_exporter'@'10.11.12.104' IDENTIFIED BY 'exporterpassword' WITH MAX_USER_CONNECTIONS 2;"
      sudo mysql -u root -prootpassword -e "GRANT PROCESS, REPLICATION CLIENT, SELECT ON *.* TO 'mysqld_exporter'@'10.11.12.104';"
      sudo mysql -u root -prootpassword -e "GRANT SELECT ON performance_schema.* TO 'mysqld_exporter'@'10.11.12.104';"
      sudo mysql -u root -prootpassword -e "FLUSH PRIVILEGES;"

      mysqldump -u root -prootpassword --all-databases --master-data > /vagrant/masterdump.sql
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
      sudo DEBIAN_FRONTEND=noninteractive apt-get install -y mysql-server

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
      curl -s https://api.github.com/repos/prometheus/mysqld_exporter/releases/latest   | grep browser_download_url   | grep linux-amd64 | cut -d '"' -f 4   | wget -qi -
      tar xvf mysqld_exporter*.tar.gz
      sudo mv  mysqld_exporter-*.linux-amd64/mysqld_exporter /usr/local/bin/
      sudo chmod +x /usr/local/bin/mysqld_exporter

      sudo tee /etc/.mysqld_exporter.cnf <<EOF
[client]
user=mysqld_exporter
password=exporterpassword
host=10.11.12.101
EOF
      sudo groupadd --system prometheus
      sudo useradd -s /sbin/nologin --system -g prometheus prometheus
      sudo chown root:prometheus /etc/.mysqld_exporter.cnf
      sudo tee /etc/systemd/system/mysql_exporter.service <<EOF
[Unit]
Description=Prometheus MySQL Exporter
After=network.target
User=prometheus
Group=prometheus

[Service]
Type=simple
Restart=always
ExecStart=/usr/local/bin/mysqld_exporter \
--config.my-cnf /etc/.mysqld_exporter.cnf \
--collect.global_status \
--collect.info_schema.innodb_metrics \
--collect.auto_increment.columns \
--collect.info_schema.processlist \
--collect.binlog_size \
--collect.info_schema.tablestats \
--collect.global_variables \
--collect.info_schema.query_response_time \
--collect.info_schema.userstats \
--collect.info_schema.tables \
--collect.perf_schema.tablelocks \
--collect.perf_schema.file_events \
--collect.perf_schema.eventswaits \
--collect.perf_schema.indexiowaits \
--collect.perf_schema.tableiowaits \
--collect.slave_status \
--web.listen-address=10.11.12.104:9104

[Install]
WantedBy=multi-user.target
      sudo systemctl daemon-reload
      sudo systemctl enable mysql_exporter
      sudo systemctl start mysql_exporter
EOF

      sudo systemctl daemon-reload
      sudo systemctl enable mysql_exporter
      sudo systemctl start mysql_exporter

      sudo apt -y install firewalld
      sudo mkdir /var/lib/prometheus

      for i in rules rules.d files_sd; do sudo mkdir -p /etc/prometheus/${i}; done

      mkdir -p /tmp/prometheus && cd /tmp/prometheus
      curl -s https://api.github.com/repos/prometheus/prometheus/releases/latest \
        | grep browser_download_url \
        | grep linux-amd64 \
        | cut -d '"' -f 4 \
        | wget -qi -

      tar xvf prometheus*.tar.gz
      cd prometheus*/
      sudo mv prometheus promtool /usr/local/bin/
      sudo mv prometheus.yml  /etc/prometheus/prometheus.yml
      sudo mv consoles/ console_libraries/ /etc/prometheus/

      cat <<EOF | sudo tee /etc/systemd/system/prometheus.service
[Unit]
Description=Prometheus
Documentation=https://prometheus.io/docs/introduction/overview/
Wants=network-online.target
After=network-online.target

[Service]
Type=simple
Environment="GOMAXPROCS=5"
User=prometheus
Group=prometheus
ExecReload=/bin/kill -HUP $MAINPID
ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/var/lib/prometheus \
  --web.console.templates=/etc/prometheus/consoles \
  --web.console.libraries=/etc/prometheus/console_libraries \
	--web.listen-address=0.0.0.0:9090 \
  --web.external-url=

SyslogIdentifier=prometheus
Restart=always

[Install]
WantedBy=multi-user.target
EOF

      for i in rules rules.d files_sd; do sudo chown -R prometheus:prometheus /etc/prometheus/${i}; done
      for i in rules rules.d files_sd; do sudo chmod -R 775 /etc/prometheus/${i}; done
      sudo chown -R prometheus:prometheus /var/lib/prometheus/

      sudo systemctl daemon-reload
      sudo systemctl start prometheus
      sudo systemctl enable prometheus

      sudo firewall-cmd --add-port=9090/tcp --permanent
      sudo firewall-cmd --add-port=9104/tcp --permanent
      sudo firewall-cmd --reload

      cat <<EOF | sudo tee /etc/.mysqld_exporter.cnf
[client]
user=mysqld_exporter
password=exporterpassword
host=10.11.12.101
EOF
      sudo chown root:prometheus /etc/.mysqld_exporter.cnf

      cat <<EOF | sudo tee /etc/systemd/system/mysql_exporter.service
[Unit]
Description=Prometheus MySQL Exporter
After=network.target
User=prometheus
Group=prometheus

[Service]
Type=simple
Restart=always
ExecStart=/usr/local/bin/mysqld_exporter \
--config.my-cnf /etc/.mysqld_exporter.cnf \
--collect.global_status \
--collect.info_schema.innodb_metrics \
--collect.auto_increment.columns \
--collect.info_schema.processlist \
--collect.binlog_size \
--collect.info_schema.tablestats \
--collect.global_variables \
--collect.info_schema.query_response_time \
--collect.info_schema.userstats \
--collect.info_schema.tables \
--collect.perf_schema.tablelocks \
--collect.perf_schema.file_events \
--collect.perf_schema.eventswaits \
--collect.perf_schema.indexiowaits \
--collect.perf_schema.tableiowaits \
--collect.slave_status \
--web.listen-address=10.11.12.104:9104

[Install]
WantedBy=multi-user.target
EOF

      sudo systemctl daemon-reload
      sudo systemctl enable mysql_exporter
      sudo systemctl start mysql_exporter

      cat <<EOF | sudo tee /etc/prometheus/prometheus.yml
# my global config
global:
  scrape_interval: 15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).

# Alertmanager configuration
alerting:
  alertmanagers:
    - static_configs:
        - targets:
          # - alertmanager:9093

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: "prometheus"

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.

    static_configs:
      - targets: ["localhost:9090"]

  - job_name: "mysql_server1"
    static_configs:
      - targets: ['10.11.12.104:9104']
        labels:
          alias: db1
EOF

      sudo systemctl restart prometheus

      sudo apt-get install -y gnupg software-properties-common
      wget -q -O /tmp/KEY.gpg https://packages.grafana.com/gpg.key
      sudo apt-key add /tmp/KEY.gpg
      echo "deb https://packages.grafana.com/oss/deb stable main" | sudo tee /etc/apt/sources.list.d/grafana.list
      sudo apt-get update
      sudo apt-get install -y grafana
      sudo systemctl start grafana-server
      sudo systemctl enable grafana-server

      # Open port for Grafana
      sudo ufw allow 3000/tcp
      sudo firewall-cmd --add-port=3000/tcp --permanent
      sudo firewall-cmd --reload
    SHELL
  end
end
