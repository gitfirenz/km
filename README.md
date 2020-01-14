# km
Mon KM

## CENTOS

### restart

shutdown -r now

## SELINUX

```
sestatus
cat /etc/selinux/config
audit2why -b -a # audit des blocages
setsebool -P nis_enabled 1 # activation d'un boolean selinux pour autoriser l'usage de NIS (cas rsyslog qui lit sur le disque)
```

## KAFKA

### Builder kafkacat sous Centos 7.7

```bash
# la base pour compiler de C++
sudo yum install gcc # le compilateur
sudo yum install cmake
sudo yum group install "Development Tools"

# les dépendances Kafka
sudo yum install -y librdkafka-devel # la base
sudo yum install -y yajl-devel # support JSON
sudo yum install -y  cyrus-sasl-devel # Support Auth SASL
sudo yum install -y libzstd-devel # Compression, cf https://github.com/facebook/zstd
sudo yum install -y libcurl-devel

cd /tmp
git clone https://github.com/edenhill/kafkacat.git
cd kafkacat
sudo ./bootstrap.sh
sudo ./bootstrap.sh # si la compilation a planté car trop de parallelisation

```

### installer kafkacat sur toute la centos

```bash
sudo mv ./kafkacat /usr/local/bin
sudo chmod 755 /usr/local/bin/kafkacat
```

### Kafka lui même

```bash
systemctl start zookeeper
systemctl start kafka

echo "coucou $(date)" | kafkacat -P -b localhost:9092 -t rsyslog
kafkacat -C -b localhost:9092 -t rsyslog

```


### /etc/systemd/system/zookeeper.service
```
[Unit]
Requires=network.target remote-fs.target
After=network.target remote-fs.target

[Service]
Type=simple
User=kafka
ExecStart=/home/kafka/kafka/bin/zookeeper-server-start.sh /home/kafka/kafka/config/zookeeper.properties
ExecStop=/home/kafka/kafka/bin/zookeeper-server-stop.sh
Restart=on-abnormal

[Install]
WantedBy=multi-user.target

```


### /etc/systemd/system/kafka.service
```
[Unit]
Requires=zookeeper.service
After=zookeeper.service

[Service]
Type=simple
User=kafka
ExecStart=/bin/sh -c '/home/kafka/kafka/bin/kafka-server-start.sh /home/kafka/kafka/config/server.properties > /home/kafka/kafka/kafka.log 2>&1'
ExecStop=/home/kafka/kafka/bin/kafka-server-stop.sh
Restart=on-abnormal

[Install]
WantedBy=multi-user.target
```


### Rsyslog pour envoyer vers Kafka (Centos 7)

Objectif : fichiers /tmp/local*.log ligne/ligne -> kafka sans logstash (perf +++)

```
sudo setsebool -P nis_enabled 1
```


### omkafka: kafka message ... Permission denied

nis_enabled 1 (Reflexe selinux : audit2why -b -a )

### Rsyslog Listener UDP -> Kafka

On va ajouter une conf supplémentaire rsyslog

```
sudo cp /usr/lib/systemd/system/rsyslog.service /usr/lib/systemd/system/rsyslog_to_kafka.service
sudo vi /usr/lib/systemd/system/rsyslog_to_kafka.service
# ExecStart=/usr/sbin/rsyslogd -n $SYSLOGD_OPTIONS -f /etc/rsyslog_to_kafka.conf -d -i /var/run/rsyslog_to_kafka.pid

sudo vi /etc/rsyslog_to_kafka.conf
sudo systemctl daemon-reload
```

#### rsyslog_to_kafka.service

/usr/lib/systemd/system/rsyslog_to_kafka.service

```
[Unit]
Description=System Logging Service
;Requires=syslog.socket
Wants=network.target network-online.target
After=network.target network-online.target
Documentation=man:rsyslogd(8)
Documentation=http://www.rsyslog.com/doc/

[Service]
Type=notify
EnvironmentFile=-/etc/sysconfig/rsyslog
ExecStart=/usr/sbin/rsyslogd -n $SYSLOGD_OPTIONS -f /etc/rsyslog_to_kafka.conf -i /var/run/rsyslog_to_kafka.pid -d
Restart=on-failure
UMask=0066
StandardOutput=null
Restart=on-failure

[Install]
WantedBy=multi-user.target
;Alias=syslog.service
```

sudo systemctl daemon-reload

#### rsyslog_to_kafka.conf

```
#### MODULES ####
module(load="omkafka")   # lets you send to Kafka

# Provides UDP syslog reception
$ModLoad imfile
$ModLoad imudp
$UDPServerRun 514

#### GLOBAL DIRECTIVES ####

# Where to place auxiliary files
$WorkDirectory /var/lib/rsyslog

# Use default timestamp format
$ActionFileDefaultTemplate json_lines

#### RULES ####
# remote host is: name/ip:port, e.g. 192.168.0.1:514, port optional
#*.* @@remote-host:514

input(type="imfile"
  File="/tmp/local*.log"
  Tag="locallogs"
)

template(name="json_lines" type="list" option.json="on") {
  constant(value="{")
  constant(value="\"timestamp\":\"")
  property(name="timereported" dateFormat="rfc3339")
  constant(value="\",\"message\":\"")
  property(name="msg")
  constant(value="\",\"host\":\"")
  property(name="hostname")
  constant(value="\",\"severity\":\"")
  property(name="syslogseverity-text")
  constant(value="\",\"facility\":\"")
  property(name="syslogfacility-text")
  constant(value="\",\"syslog-tag\":\"")
  property(name="syslogtag")
  constant(value="\"}")
}


main_queue(
  queue.workerthreads="3"      # threads to work on the queue
  queue.dequeueBatchSize="1000" # max number of messages to process at once
  queue.size="10000"           # max queue size
)


action(
  broker=["dev:9092"]
  type="omkafka"
  topic="rsyslog"
  template="json_lines"
)

```

sudo systemctl restart rsyslog_to_kafka
sudo systemctl status rsyslog_to_kafka


