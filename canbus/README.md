# CAN Bus

This is README for CAN Bus project

## Getting Started

These instructions will get you a copy of the project up and running on your local machine for development and testing purposes. 

### Prerequisites

* [Ansible](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html#installing-ansible-on-rhel-centos-or-fedora)
* [Terraform](https://learn.hashicorp.com/terraform/getting-started/install.html)
* [git](https://gist.github.com/derhuerst/1b15ff4652a867391f03#file-linux-md)
* [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html)

### Spin the environment in AWS using Terraform modules

- Ensure we have a default AWS profile set. If you want to use a named profile, you'll have to specify that when spinning up the environment.
```
$ aws configure list 
      Name                    Value             Type    Location
      ----                    -----             ----    --------
   profile                <not set>             None    None
access_key     ****************4I4K shared-credentials-file    
secret_key     ****************Wx4j shared-credentials-file    
    region                us-west-1      config-file    ~/.aws/config
```
- Clone `tf-mini-iaas` git repository. We have all the files we need in [manoj](https://github.com/Toyota-ITC/tf-mini-iaas/tree/master/manoj) directory
- Change variables in `terraform.tfvars` file. Important ones are:
```
AMI = "ami-02eac2c0129f6376b" # Choose something that gives latest CentOS 7 with EBS and ENA in the region
# AWS region to create the resources at
AWS_REGION = "us-east-1"
AWS_SUBNET_AZ = "us-east-1a"
```
- Spin the environment 
```
terraform init
terraform plan -out <plan file>
terraform apply <plan file>
```

### Configure instances using Ansible

- Checkout `development` branch from [nttd_playbook](https://github.com/Toyota-ITC/nttd_playbook) repository
- Ensure we have the [inventory file](https://github.com/Toyota-ITC/nttd_playbook/blob/development/proto1_infra/ansible/hosts/aws-itc/hosts) updated
- Apply ansible playbooks in the following sequence
    - Generator
        1. `cd <Root Directory>/proto1_infra/ansible`
        1. `ansible-playbook -i hosts/aws-itc/hosts -l generator osmount.yml`
        1. `ansible-playbook -i hosts/aws-itc/hosts -l generator generator.yml`
        1. `ansible -i hosts/aws-itc/hosts all -l generator -m shell -a "shutdown -r now"`
        1. `ansible-playbook -i hosts/aws-itc/hosts -l generator operation/disk/disk_mount.yml`
    - Annotation
        1. `cd <Root Directory>/proto1_infra/ansible`
        1. `ansible-playbook -i hosts/aws-itc/hosts -l receive_annotation osmount.yml`
        1. `ansible-playbook -i hosts/aws-itc/hosts annotation.yml`
        1. `ansible -i hosts/aws-itc/hosts all -l receive_annotation -m shell -a "shutdown -r now"`
        1. `ansible-playbook -i hosts/aws-itc/hosts -l receive_annotation operation/disk/disk_mount.yml`
        1. `ansible-playbook -i hosts/aws-itc/hosts -l receive_annotation operation/service/mw/service-start.yml`
    - Decode
        1. `cd <Root Directory>/proto1_infra/ansible`
        1. `ansible-playbook -i hosts/aws-itc/hosts -l phyqtyconv osmount.yml`
        1. `ansible-playbook -i hosts/aws-itc/hosts phyqtyconv.yml`
        1. `ansible-playbook -i hosts/aws-itc/hosts -l "phy*" operation/hdfs/zkfc_format.yml`
        1. `ansible-playbook -i hosts/aws-itc/hosts -l "phy*" operation/hdfs/nn_format.yml`
        1. `ansible-playbook -i hosts/aws-itc/hosts -l "phy*" operation/service/mw/service-stop.yml --skip-tags "history-server-service"`
        1. `ansible-playbook -i hosts/aws-itc/hosts -l "phy*" operation/service/mw/service-start.yml --skip-tags "history-server-service"`
        1. `ansible-playbook -i hosts/aws-itc/hosts -l "phy*" operation/hdfs/nn_failover.yml`
        1. `ansible-playbook -i hosts/aws-itc/hosts -l "phy*" operation/hdfs/make_hdfs_dir.yml`
        1. `ansible-playbook -i hosts/aws-itc/hosts -l "phy*" operation/service/mw/service-start.yml --tags "history-server-service"`
        1. `ansible-playbook -i hosts/aws-itc/hosts -l "phy*" operation/service/mw/service-stop.yml`
        1. `ansible -i hosts/aws-itc/hosts all -l "phy*" -m shell -a "shutdown -r now"`
        1. `ansible-playbook -i hosts/aws-itc/hosts -l "phy*" operation/disk/disk_mount.yml`
        1. `ansible-playbook -i hosts/aws-itc/hosts -l "phy*" operation/service/mw/service-start.yml`
        1. `ansible-playbook -i hosts/aws-itc/hosts -l "phy*" operation/hdfs/nn_failover.yml`
    - Rawstore
        1. `cd <Root Directory>/proto1_infra/ansible`
        1. `ansible-playbook -i hosts/aws-itc/hosts -l rawstore osmount.yml`
        1. `ansible-playbook -i hosts/aws-itc/hosts rawstore.yml`
        1. `ansible-playbook -i hosts/aws-itc/hosts -l "rawstore*" operation/hdfs/zkfc_format.yml`
        1. `ansible-playbook -i hosts/aws-itc/hosts -l "rawstore*" operation/hdfs/nn_format.yml`
        1. `ansible-playbook -i hosts/aws-itc/hosts -l "rawstore*" operation/service/mw/service-stop.yml --skip-tags "history-server-service","hive-service"`
        1. `ansible-playbook -i hosts/aws-itc/hosts -l "rawstore*" operation/service/mw/service-start.yml --skip-tags "history-server-service","hive-service"`
        1. `ansible-playbook -i hosts/aws-itc/hosts -l "rawstore" operation/hdfs/nn_failover.yml`
        1. `ansible-playbook -i hosts/aws-itc/hosts -l "rawstore" operation/hdfs/make_hdfs_dir.yml`
        1. `ansible-playbook -i hosts/aws-itc/hosts -l "rawstore*" operation/hdfs/spark16_sparkjob_put.yml`
        1. `ansible-playbook -i hosts/aws-itc/hosts hive_init.yml`
        1. `ansible-playbook -i hosts/aws-itc/hosts -l "rawstore*" operation/service/mw/service-start.yml --tags "history-server-service","hive-service"`
        1. `ansible-playbook -i hosts/aws-itc/hosts -l "rawstore*" operation/service/mw/service-stop.yml`
        1. `ansible -i hosts/aws-itc/hosts all -l rawstore -m shell -a "shutdown -r now"`
        1. `ansible-playbook -i hosts/aws-itc/hosts -l "rawstore*" operation/disk/disk_mount.yml`
        1. `ansible-playbook -i hosts/aws-itc/hosts -l "rawstore*" operation/service/mw/service-start.yml`
        1. `ansible-playbook -i hosts/aws-itc/hosts -l "rawstore*" operation/hdfs/nn_failover.yml`
  
### Deploy the application using Ansible

- Ensure we have the [inventory file](https://github.com/Toyota-ITC/nttd_playbook/blob/development/proto1_infra/ansible_app/hosts/aws-itc/hosts) updated
- Apply ansible playbooks in the following sequence
    - Generator
        1. `cd <Root Directory>/proto1_infra/ansible_app`
        1. `ansible-playbook -i hosts/aws-itc/hosts generator.yml`
    - Amplifier
        1. `cd <Root Directory>/proto1_infra/ansible_app`
        1. `ansible-playbook -i hosts/aws-itc/hosts amplifier.yml`
    - Annotation
        1. `cd <Root Directory>/proto1_infra/ansible_app`
        1. `ansible-playbook -i hosts/aws-itc/hosts annotation.yml`      
    - Decode
        1. `cd <Root Directory>/proto1_infra/ansible_app`
        1. `ansible-playbook -i hosts/aws-itc/hosts -l phyqtyconv-* phyqtyconv.yml` # Decode Streaming
        1. `ansible-playbook -i hosts/aws-itc/hosts -l rawstore-* phyqtyconv.yml`   # Decode Batch
    - Rawstore
        1. `cd <Root Directory>/proto1_infra/ansible_app`
        1. `ansible-playbook -i hosts/aws-itc/hosts rawstore.yml`   # Rawstore Streaming          
        
### Start the Applications
- Checkout `development` branch from [nttd_playbook](https://github.com/Toyota-ITC/nttd_tool) repository
- Ensure `config.json` file has the hosts that we have created
- Application start sequence is as follows: Annotator, Amplifier, Generator
- Application stop sequence is as follows: Generator, Amplifier, Annotator

#### Prerequisites

Create a topic `candata-raw-annotated-shisaku` with 512 partitions, replication factor of 3, if it doesn't exist on Rawstore Kafka cluster. For test purposes, there's no need to enable compression for the topic.

```
ssh <rawstore broker host>
/opt/kafka/bin/kafka-topics.sh --zookeeper itc-rawzookeeper01:2181 --create --topic candata-raw_annotated-shisaku01 --partitions 512 --replication-factor 3 
```

Create a topic `candata-decoded-shisaku01` with 512 partitions, replication factor of 3, if it doesn't exist on Rawstore Kafka cluster. For test purposes, there's no need to enable compression for the topic.

```
ssh <decode broker host>
/opt/kafka/bin/kafka-topics.sh --zookeeper itc-phyzookeeper01:2181 --create --topic candata-decoded-shisaku01 --partitions 512 --replication-factor 3 
```

#### Annotator

Before starting the Application for the first time, input control data in Redis cluster by executing following command. These steps need to be done from `nttd_tool` project root. Following step adds Kafka specific data (brokers, topic name) in Redis.

```
cd <Root Directory> 
./inputDataAnnDB.sh 
```

Start the Annotator by running following command.

```
cd <Root Directory> 
# Protocol
PROTOCOL="http"
# Inventory
config_json="$(dirname $0)/config.json"
./shell/ann/startAnn.sh "${config_json}" $PROTOCOL postgres
```

#### Amplifier

```
cd <Root Directory> 
# Amplification rate (10,30)
DATA_MAG=30
# Protocol
PROTOCOL="http"
# Use Load Balancer (1 for Yes, 0 for No)
lbFlg=1
# Inventory
config_json="$(dirname $0)/config.json"
./shell/amp/startAmp.sh "${config_json}" $DATA_MAG $PROTOCOL $lbFlg
```

#### Generator

Make a directory in generator host and copy file used by the script.

```
ssh <generator host>
sudo mkdir -p /home/centos/git/nttd_tool/shell/gen; sudo chown centos:centos /home/centos/git/nttd_tool/shell/gen
# Copy file from nttd_tool
scp <Root Directory>/shell/gen/createSettingTxt.sh <generator host>:/home/centos/git/nttd_tool/shell/gen
```

Start the generator using the following script:

```
# Generator operation type
VAR="redu"
# Number of vehicles (all servers)
VEHICLE_NUM=6000
# Transmission interval, When the transmission interval is 60 seconds, it is assumed that 100 TPS without amplification (1x).
INTERVAL=60
# Amplification rate (2/3/8/12) Affects message size
DATA_SIZE=3
# Execution time (minutes)
# If you specify a long time, it will end as soon as the original data is transmitted.
# The length of the original data is about 15 minutes.
TIMEOUT=15
# Protocol
PROTOCOL="http"
# Inventory
config_json="$(dirname $0)/config.json"

./shell/gen/startGen.sh "${config_json}" ${VAR} ${VEHICLE_NUM} ${INTERVAL} ${DATA_SIZE} ${TIMEOUT} ${PROTOCOL}

```

#### RawStore Streaming Application

This is a spark streaming application. This job reads data from `candata-raw-annotated-shisaku01` kafka topic and writes to HDFS.

Login to one of the rawstore master hosts. Check `/opt/rawstore/conf/rawstore_cluster.properties` file to ensure we have right values. Following lists properties that were tested in ITC AWS environment. Make sure the HDFS paths are read/writeble by the user that's running the Application.

```
#Spark input information
rawstore.streaming.checkpoint=hdfs://hdfs-cluster/user/nttuser01/checkpoint/rawstore/shisaku01
rawstore.streaming.interval=60000

#kafka input information
rawstore.input.kafka.url=itc-rawbroker04:9092,itc-rawbroker02:9092,itc-rawbroker03:9092,itc-rawbroker01:9092
rawstore.input.kafka.topic=candata-raw-annotated-shisaku01
rawstore.input.kafka.group.id=rawstore

#RawStore output information (SequenceFile only)
#rawstore.output.sequenceFileDelimiter=ffffffffffffffffffff

#RawStore input information
rawstore.output.numPartitions=512
rawstore.output.rootDirectory=hdfs://hdfs-cluster/candata/raw_annotated/model=shisaku01

#RawStore class information
rawstore.function.createStream=jp.co.nttdata.ccfw.acc.rawstore.kafka.CreateStreamImpl
rawstore.function.extractFromBinaryFunction=jp.co.nttdata.ccfw.acc.rawstore.function.ExtractFromBinaryFunctionImpl
rawstore.function.filterFunction=jp.co.nttdata.ccfw.acc.rawstore.function.FilterFunctionImpl
rawstore.function.rawStoreWriterFunction=jp.co.nttdata.ccfw.acc.rawstore.function.ParquetWriterFunctionImpl

#kafka consumer parameter(Optional)
rawstore.input.kafka.additional["request.timeout.ms"]=3050000
```

Start the application using following command on one of the rawstore master hosts:

```
/opt/rawstore/bin/start_rawstore.sh rawStore rawstore_cluster.properties
```

#### Decode Batch Application

This is a Spark Streaming Application. It reads data from HDFS that is created by Rawstore Streaming Application and writes it to HDFS location on Raw Hadoop cluster. Once done, this job finishes (unlike Rawstore Streaming Application, which keeps running until killed explicitly).

Login to one of the rawstore master hosts. Check ` /opt/phyqtyconv/conf/physical-data-converter-batch.properties` file to ensure we have right values. Following lists properties that were tested in ITC AWS environment. Make sure the HDFS paths are read/writeble by the user that's running the Application.

```
$ cat /opt/phyqtyconv/conf/physical-data-converter-batch.properties 
input.parquet.path = hdfs://hdfs-cluster/candata/raw_annotated/model=shisaku01
input.sequence.path = hdfs://hdfs-cluster/candata/raw_annotated/model=shisaku01
input.sequence.delimiter = ffffffffffffffffffff
input.sequence.message.position = 3
input.sequence.message.isLast = true

output.parquet.path = hdfs://hdfs-cluster/phys_conv_output/shisaku01/par/
output.sequence.path = hdfs://hdfs-cluster/phys_conv_output/shisaku01/seq/
output.avro.path = hdfs://hdfs-cluster/phys_conv_output/shisaku01/avro/

dsl.amp.factor = 3
```
Start the application using following command on one of the rawstore master hosts:

```
$ /opt/phyqtyconv/bin/start_phyqtycnv_batch.sh ParquetToParquet physConvBatch physical-data-converter-batch.properties dsl-model_b-1.0-SNAPSHOT.jar 5 1 1G 4G sp20
```
#### Decode Streaming Application

This is a Spark Streaming Application. It reads annotated data from Kafka topic in Rawstore kafka cluster, does some decoding and writes data to another topic in Decode Kafka cluster, based on configuration. 

Login to one of the decode master hosts. Check `/opt/phyqtyconv/conf/physical-data-converter-stream.properties ` file to ensure we have right values. Following lists properties that were tested in ITC AWS environment. Make sure the HDFS paths are read/writeble by the user that's running the Application.

```
$ cat /opt/phyqtyconv/conf/physical-data-converter-stream.properties 
input.kafka.url=itc-rawbroker01:9092,itc-rawbroker02:9092,itc-rawbroker03:9092,itc-rawbroker04:9092
input.kafka.topic=candata-raw_annotated-shisaku01
input.kafka.group.id=physical-data-converter

output.kafka.url=itc-phybroker01:9092,itc-phybroker02:9092,itc-phybroker03:9092
output.kafka.topic=candata-decoded-shisaku01

output.kafka.acks=all
output.kafka.batch.size=16384
output.kafka.linger.ms=0
output.kafka.compression.type=none

streaming.interval=10000
streaming.checkpoint=hdfs://hdfs-cluster/user/nttuser01/checkpoint/phyqtyconv/shisaku01

window.type=NUM
window.length=3
window.slideInterval=1
window.futureScopeLength=1
window.pastScopeLength=1

window.numberWindowTimeout=180000
#window.numberWindowCheckpointInterval=100000
window.numberWindowRedisAddresses=172.31.32.121:6379,172.31.32.121:6380,172.31.32.121:6381,172.31.32.121:6382,172.31.32.121:6383,172.31.32.121:6384,172.31.32.121:6385,172.31.32.121:6386,172.31.32.122:6379,172.31.32.122:6380,172.31.32.122:6381,172.31.32.122:6382,172.31.32.122:6383,172.31.32.122:6384,172.31.32.122:6385,172.31.32.122:6386,172.31.32.123:6379,172.31.32.123:6380,172.31.32.123:6381,172.31.32.123:6382,172.31.32.123:6383,172.31.32.123:6384,172.31.32.123:6385,172.31.32.123:6386
```

Start the application using following command on one of the decode master hosts:

```
/opt/phyqtyconv/bin/start_phyqtycnv_stream.sh decodeStreaming physical-data-converter-stream.properties dsl-model_b_60-1.0-SNAPSHOT.jar 60 4 1
```


### Stop the Applications

#### Decode Streaming Application

Kill the YARN Application from one of the decode master hosts.

```
yarn application -kill <App ID> 
```

#### RawStore Batch Application

This job finishes by itself once it is done, so killing is not necessary.

#### RawStore Streaming Application

Kill the YARN Application from one of the raw master hosts.

```
yarn application -kill <App ID> 
```

#### Generator

Run the following script from the root directory of `nttd_tool` project.

```
config_json="$(dirname $0)/config.json"
./shell/gen/stopGen.sh "${config_json}"
```

#### Amplifier

Run the following script from the root directory of `nttd_tool` project.

```
config_json="$(dirname $0)/config.json"
./shell/gen/stopAmp.sh "${config_json}"
```

#### Annotator

Run the following script from the root directory of `nttd_tool` project.

```
config_json="$(dirname $0)/config.json"
./shell/gen/stopGen.sh "${config_json}"
```










