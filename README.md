# Local MySQL Cluster with Monitoring via Grafana
Example MySQL cluster in a local environment

To start:
`vagrant up`
Go to `http://10.11.12.104:3000/` and login
Add the Prometheus data source from `http://localhost:9090`
Import the MySQL Server Exporter dashboard ID `14057`
`vagrant ssh mysql-master`
`mysql -u root -prootpassword`
`create user sysadmin identified by 'mypassword';`
`grant all on *.* to sysadmin;`
`quit;`
`mkdir mysqlslap_tutorial`
`cd mysqlslap_tutorial`
`sudo wget https://launchpad.net/test-db/employees-db-1/1.0.6/+download/employees_db-full-1.0.6.tar.bz2`
`sudo bzip2 -dfv employees_db-full-1.0.6.tar.bz2`
`sudo tar -xf employees_db-full-1.0.6.tar`
`cd employees_db`
`sed -i 's/storage_engine/default_storage_engine/g' employees.sql`
`sudo mysql -h localhost -u sysadmin -p -t < employees.sql`
`sudo mysqlslap --user=sysadmin --password --host=localhost  --auto-generate-sql --verbose`
`sudo mysqlslap --user=sysadmin --password --host=localhost  --concurrency=50 --iterations=10 --auto-generate-sql --verbose`
`sudo mysqlslap --user=sysadmin --password --host=localhost  --concurrency=50 --iterations=100 --number-int-cols=5 --number-char-cols=20 --auto-generate-sql --verbose`
`exit`
`vagrant halt`
`vagrant destroy -f`

