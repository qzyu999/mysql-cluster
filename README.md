# Local MySQL Cluster with Monitoring via Grafana
Example MySQL cluster in a local environment

Main references:
- [https://www.linuxtrainingacademy.com/mysql-master-slave-replication-ubuntu-linux/](https://www.linuxtrainingacademy.com/mysql-master-slave-replication-ubuntu-linux/)
- [https://help.scalegrid.io/docs/mysql-connecting-to-prometheus-and-grafana](https://help.scalegrid.io/docs/mysql-connecting-to-prometheus-and-grafana)
- [https://www.youtube.com/watch?v=q2C1K69pRBY](https://www.youtube.com/watch?v=q2C1K69pRBY)
- [https://www.youtube.com/watch?v=jc4D7g1xuYg](https://www.youtube.com/watch?v=jc4D7g1xuYg)
- [https://www.youtube.com/watch?v=cjIb-lKeN5s](https://www.youtube.com/watch?v=cjIb-lKeN5s)
- [https://www.digitalocean.com/community/tutorials/how-to-measure-mysql-query-performance-with-mysqlslap](https://www.digitalocean.com/community/tutorials/how-to-measure-mysql-query-performance-with-mysqlslap)
- [https://grafana.com/oss/prometheus/exporters/mysql-exporter/?tab=dashboards](https://grafana.com/oss/prometheus/exporters/mysql-exporter/?tab=dashboards)

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

`sudo sed -i 's/storage_engine/default_storage_engine/g' employees.sql`

`sudo mysql -h localhost -u sysadmin -p -t < employees.sql`

`sudo mysqlslap --user=sysadmin --password --host=localhost  --auto-generate-sql --verbose`

`sudo mysqlslap --user=sysadmin --password --host=localhost  --concurrency=50 --iterations=10 --auto-generate-sql --verbose`

`sudo mysqlslap --user=sysadmin --password --host=localhost  --concurrency=50 --iterations=100 --number-int-cols=5 --number-char-cols=20 --auto-generate-sql --verbose`

`exit`

`vagrant halt`

`vagrant destroy -f`

To do:
- Have the mysqld_exporter listen on all database nodes (not just master).
- Make a secure way to handle secrets.
- Tune the CPU/memory allocation for the nodes in the cluster.
- Address error messages in Vagrant logs.