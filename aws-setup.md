# AWS setup

* Sign into AWS and create a new IAM role for a programmatic user with `AmazonEC2FullAccess` credentials
* Grab the Access Key and Secret Key as you'll need them when using docker-machine
* make sure that the VPC used contains security group named `docker-machine` and that it has the following ports open (see https://gist.github.com/BretFisher/7233b7ecf14bc49eb47715bbeb2a2769):
  * TCP port 2377 for cluster management communications
  * TCP and UDP port 7946 for communication among nodes
  * TCP and UDP port 4789 for overlay network traffic
