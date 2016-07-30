# Detailed log of how to make this work (on Mac OS X)

This is using commands for `eu-central-1` (Frankfurt).

## add a key dcos-centos in AWS

* Go to [AWS keypair page](https://eu-central-1.console.aws.amazon.com/ec2/v2/home?region=eu-central-1#KeyPairs:sort=keyName)
* Make a `dcos-centos` keypair
* download the pem file and store it safely + copy it in the `keys` subdirectory of this repo
* execute `chmod 400 dcos-centos` on it
* start ssh agent : `ssh-agent`
* `ssh-add` this key
* `ssh-add -l` to validate that it is in the list

## Set-up access to proper AWS IAM user

Either in .zshrc or .bashrc or in the terminal,
export the AWS env variables:

```
cd && cd p/dcos-up
export AWS_ACCESS_KEY_ID=AKIA...2A
export AWS_SECRET_ACCESS_KEY=6e...
export AWS_DEFAULT_REGION="eu-central-1"
env | grep AWS # to validate if it worked
```

## Install the cluster (`terraform apply`)

* `terraform -v` => `Terraform v0.6.16`
* if not present, [install terraform](https://www.terraform.io/intro/getting-started/install.html)
* validate on AWS that no ec2 nodes are running yet: [AWS dashboard](https://eu-central-1.console.aws.amazon.com/ec2/v2/home?region=eu-central-1#)
* also remove left-over EBS Volumes if relevant (easier adn safer now than later)
* maybe adapt the size of the machines (I use m4.xlarge which works for small demos)

```
variable "instance_types" {
  default = {
    bootstrap    = "m4.xlarge"
    master       = "m4.xlarge"
    slave        = "m4.xlarge"
    slave_public = "m4.xlarge"
  }
}
```

* run `terraform destroy` (to make sure context is clean)
* inspect `terraform.tfstate` => should be essentially empty
* run `terraform apply` (in my set-up it took 20 minutes to bring up the cluster)
  * particularly the `yum upgrade` of dcos_bootstrap takes some time ...
  (a few minutes for 122 packages to upgrade)
  * and then in parallel for all other nodes as well
  * a more up-to-date *ami* might be useful
  * a WARNING flies by about a missing GPG Key:

```
 Importing GPG key 0x2C52609D:
   Userid     : "Docker Release Tool (releasedocker) <docker@docker.com>"
   Fingerprint: 5811 8e89 f3a9 1289 7c07 0adb f762 2157 2c52 609d
   From       : https://yum.dockerproject.org/gpg
```

  * consul 0.6.4 is installed

* use a browser to go the `dcos_ui_address` (e.g. `http://52.28.1....`)
* sometimes I see the address is immediately pingable, but it takes a
  number of minutes before the UI is actually visible (reason unclear)

* Sign-in with OAuth (I used "Sign in with Google" where I had a Google
  session open in another TAB in the same browser)

* Now you should see the `DC/OS` Dashboard:
![DC/OS Dashboard](https://github.com/petervandenabeele/dcos-up/blob/master/howto/dcos-dashboard.png "DC/OS Dashboard")

## Install the dcos cli

* Bottom left on the dashboard, click open the menu
* click on install cli; you will see a command of the form

```
# do NOT run this straight away
curl https://downloads.dcos.io/binaries/cli/darwin/x86-64/0.4.10/dcos > /usr/local/bin/dcos
chmod +x /usr/local/bin/dcos &&
dcos config set core.dcos_url http://52.xxx.xxx.xxx &&
dcos
```

* create a separate dcos directory somewhere else

```
mkdir data/projects/dcos/dcos-`date "+%Y%m%d"`
cd data/projects/dcos/dcos-`date "+%Y%m%d"`
```

* instead of running all at once, try running them one by one ...

```
curl https://downloads.dcos.io/binaries/cli/darwin/x86-64/0.4.10/dcos > /usr/local/bin/dcos &&
chmod +x /usr/local/bin/dcos
# => this always works
```

* Set the dcos_url (this now is an address without SSL)
```
dcos config set core.dcos_url http://52.xxx.xxx.xxx
dcos config show
# => This also works
```

## Login to dcos

* get the dcos login token:

```
dcos auth login
```

You need to go to an address (without SSL now), login again with Google or other OAuth
form and you get a token to copy. Enter this and the result is "login successful".

If there are problems with SSL verification, although the setting claims
`core.ssl_verify=false` it may still be needed to type `export DCOS_SSL_VERIFY=false`
to make it work (the config parameters where inconsistent at one point in time).

## Getting ssh to the nodes

* there are some tricky aspects here in this set-up

* Validate that ssh-agent has the dcos-centos key

```
ssh-add -l | grep dcos-centos
# => 2048 SHA256:....... /Users/peter_v/data/github/petervandenabeele/dcos-up/keys/dcos-centos.pem (RSA)
```

* run `dcos node ssh --leader` to see what it tries to do:
```
dcos node ssh --leader --master-proxy
# => Running `ssh -A -t core@172.xxx.xxx.xxx ssh -A -t core@172.31.7.152 `
```

* this will fail, because the users in this set-up are centos (and not core) and also, it tries to
connect to 172.xxx.xxx.xxx. while it should actually try to connect to the public address of the
mesos master. Replace `core` with `centos` and use the public address of the master node that
displays the UI:

```
ssh -A -t centos@52.xxx.xxx.xxx
# => this should work and up on the master node (with hostname ip-172-31-7-152)
```

To log onto an agent or other node, look up the _internal_ ip of an agent with
```
dcos node
#   HOSTNAME          IP                          ID
# 172.31.15.126  172.31.15.126  9e568c4a-f0b5-4674-831d-65153232767f-S2
# 172.31.4.93    172.31.4.93    9e568c4a-f0b5-4674-831d-65153232767f-S3
# 172.31.7.192   172.31.7.192   9e568c4a-f0b5-4674-831d-65153232767f-S0
# 172.31.7.204   172.31.7.204   9e568c4a-f0b5-4674-831d-65153232767f-S1
```

and then issue e.g.

```
ssh -A -t centos@52.xxx.xxx.xxx ssh -A -t centos@172.31.7.192
```

## Installing Kafka

```
dcos package describe kafka  # at this moment 1.1.9-0.10.0.0

cat kafka.json # make this file to configure kafka set-up
#{
#    "service": {
#        "name": "kafka",
#        "placement_strategy": "NODE"
#    },
#    "brokers": {
#        "count": 3,
#        "mem": 1000,
#        "disk": 1000
#    },
#
#    "kafka": {
#        "delete_topic_enable": true,
#        "log_retention_hours": 128
#    }
}

dcos package uninstall --cli kafka # old versions may be here

dcos package install --cli kafka
# Installing CLI subcommand for package [kafka] version [1.1.9-0.10.0.0]
# New command available: dcos kafka

dcos package install --app --options=kafka.json kafka
# Installing Marathon app for package [kafka] version [1.1.9-0.10.0.0]
# DC/OS Kafka Service is being installed.

dcos help kafka  # pure `dcos kafka` does not return the help screen
dcos kafka config target # to see config details
```

Looking at the connections:
```
dcos kafka connection
{
    "address": [
        "172.31.7.192:9466",
        "172.31.7.204:9872",
        "172.31.4.93:9887"
    ],
    "dns": [
        "broker-0.kafka.mesos:9466",
        "broker-1.kafka.mesos:9872",
        "broker-2.kafka.mesos:9887"
    ],
    "vip": "broker.kafka.l4lb.thisdcos.directory:9092",
    "zookeeper": "master.mesos:2181/dcos-service-kafka"
}
```

NOTE: relevant aspect is that the Zookeeper location is not under a `kafka` dir
but a `dcos-service-kafka` directory (different from my expectation).

## Running a console producer and consumer and performance tests

* Using the DCOS 1.8 tutorial now works :-) (I could only get this to work with some hacks
on the 1.7 version)
[https://dcos.io/docs/1.8/usage/tutorials/kafka/](https://dcos.io/docs/1.8/usage/tutorials/kafka/)

* first make some topics from the CLI:

```
dcos kafka topic create topic1 --partitions 1 --replication 1
dcos kafka topic create topic3 --partitions 4 --replication 3
dcos kafka topic describe topic3 # this yields an interesting list of partitions, replications, isr etc
```

* As the tutorial indicates, ssh into the leader and start the `mesosphere/kafka-client` image

```
ssh -A -t centos@52.xxx.xxx.xxx
./kafka-topics.sh --list --zookeeper master.mesos:2181/dcos-service-kafka # NOTE different location in Zookeeper !
# topic1
# topic3
```

* Console producers and consumers work :-)

```
docker run -it mesosphere/kafka-client
# wait for docker container to load and to come up with
# root@352e09038427:/bin#
./kafka-console-producer.sh --broker-list 172.31.7.192:9466,172.31.7.204:9872,172.31.4.93:9887 --topic topic1
foo
bar
# Control-D (to stop writing)
./kafka-console-consumer.sh --zookeeper master.mesos:2181/dcos-service-kafka --topic topic1 --from-beginning
# foo
# bar
# Control-C to stop consuming
# Processed a total of 2 messages
./kafka-console-producer.sh --broker-list 172.31.7.192:9466,172.31.7.204:9872,172.31.4.93:9887 --topic topic3
foo3
bar3
# Control-D (to stop writing)
root@352e09038427:/bin# ./kafka-console-consumer.sh --zookeeper master.mesos:2181/dcos-service-kafka --topic topic3 --from-beginning
bar3
foo3
# Control-C to stop consuming
Processed a total of 2 messages
```

* some simplistic performance tests

```
# Writing to topic3 (with 3 times replication, over all brokers) => 173 K messages per second
root@352e09038427:/bin# ./kafka-producer-perf-test.sh --topic topic3 --num-records 400000 --record-size 150 --throughput 1000000 --producer-props bootstrap.servers=172.31.7.192:9466
400000 records sent, 173385.348938 records/sec (24.80 MB/sec), 65.24 ms avg latency, 568.00 ms max latency, 14 ms 50th, 494 ms 95th, 550 ms 99th, 565 ms 99.9th.

# Reading from topic3 using the new consumer => 275 K messages per second
root@352e09038427:/bin# ./kafka-consumer-perf-test.sh --messages 400000 --broker-list 172.31.7.192:9466,172.31.7.204:9872,172.31.4.93:9887 --zookeeper master.mesos:2181/dcos-service-kafka --topic topic3 --new-consumer
start.time, end.time, data.consumed.in.MB, MB.sec, data.consumed.in.nMsg, nMsg.sec
2016-07-31 16:41:55:676, 2016-07-31 16:41:57:161, 58.5865, 39.4522, 409551, 275791.9192

# Writing to topic41 (4 partitions, replication only 1) => 261 K messages per second
root@352e09038427:/bin# ./kafka-producer-perf-test.sh --topic topic41 --num-records 400000 --record-size 150 --throughput 1000000 --producer-props bootstrap.servers=172.31.7.192:9466
400000 records sent, 261096.605744 records/sec (37.35 MB/sec), 8.75 ms avg latency, 199.00 ms max latency, 4 ms 50th, 45 ms 95th, 60 ms 99th, 62 ms 99.9th.

# Reading from topic41 (4 partitions, replication only 1) => 300 K messages per second
root@352e09038427:/bin# ./kafka-consumer-perf-test.sh --messages 400000 --broker-list 172.31.7.192:9466,172.31.7.204:9872,172.31.4.93:9887 --zookeeper master.mesos:2181/dcos-service-kafka --topic topic41 --new-consumer
start.time, end.time, data.consumed.in.MB, MB.sec, data.consumed.in.nMsg, nMsg.sec
2016-07-31 16:48:58:106, 2016-07-31 16:48:59:438, 57.2205, 42.9583, 400000, 300300.3003
```
