# km
Mon KM

## KAFKA

### Builder kafkacat sous Centos 7.7

```
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
sudo ./bootstrap.sh # si la compilation aplanté car trop de parallelisation
# 
```
