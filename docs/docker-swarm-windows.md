# Setup docker swarm
Follow this guide to set up a docker swarm on Windows 10.

We will setup a swarm of three nodes (one manager and two workers).

## Requirements
- Install Docker Desktop from here: [Docker Desktop for Windows 10](https://docs.docker.com/docker-for-windows/install/)
- Install Docker Machine by following the instructions here: [Install Docker Machine](https://docs.docker.com/machine/install-machine/)
- Install VirtualBox (for running a multi-node swarm locally) from here: [VirtualBox binaries](https://www.virtualbox.org/wiki/Downloads)

## From Host
```
for index in node{0..2}; do docker-machine create --driver virtualbox $index; done
```
This will create three nodes; node0, node1, and node2.
```
docker-machine ls
```

#### If you encounter the following error:
##### Error with pre-create check: "This computer doesn't have VT-X/AMD-V enabled. Enabling it in BIOS is mandatory"
#### Either enable VT-X or AMD-V in BIOS or pass `--virtualbox-no-vtx-check`

#### For other options available for the create command: [docker-machine create](http://docs.docker.oeynet.com/machine/reference/create/)

## Initialize the Swarm
```
docker-machine ssh node0
ip a s
```
Copy the IP address under the eth1. This is the host machine's network card as a virtual interface inside the VM.
```
docker swarm init --advertise-addr [COPIED_IP_ADDRESS]
```
This will initialize the swarm and assign the current node as a manager. A join-token will be printed which is required to add worker nodes to this swarm. Copy the complete command that is printed.

## Add Worker Nodes
Logout out from node0 by running `logout`.
```
docker-machine ssh node1
```
Paste the command copied earlier to join as a worker node. We can not run any swarm commands from a worker node.
```
logout
docker-machine ssh node2
```
Paste the same command again to join as a worker node.

To verify the swarm formation:
```
logout
docker node ls
```
This should display all nodes as Ready and ACTIVE. node0 should also have the Leader status.

## Modify Swarm State
We can add and remove managers from the swarm by promoting and demoting the existing nodes. For example, to change the manager from node0 to node1:
```
docker node promote node1
docker node demote node0
```
A swarm must have one or more managers at any time.

## Removing A Node from The Swarm
Before removing a node, we must make sure that it is not running any tasks assigned by the swarm. This is called draining the node. We can drain a node by changing its availability from active to drain.
```
# From a manager node
docker node update --availability drain node2
logout

docker-machine ssh node2
docker swarm leave
logout

# Assuming node0 is a manager
docker-machine ssh node0
# This is optional
docker node rm node2
logout

# From host machine
docker-machine stop node2
```

## Connecting to Remote Docker Runtime
We can use our host docker CLI to run against remote docker daemon, so we can avoid SSH into remote machines. For Windows 10, we must specify the shell we are using (bash, powershell, or cmd).
```
docker-machine env node0 --shell bash
```
This prints a command we can run to connect to node0's docker runtime.

To connect to our host runtime again:
```
docker-machine env --unset
```

## Run A Service on The Swarm
Services can be run in either replicated (on a fixed number of nodes) mode or global mode. We can update a running service from a manager node.
```
docker service create --name myservice nginx
```
This will pull the latest image of nginx (if not found locally) and start a service called `myservice` with 1 replica running.
```
docker service ls
docker service ps myservice
```

## Update A Running Service
To scale the service up or down (only in replicated mode), we can provide scale factor for the service.
```
docker service scale myservice=2
```
To map the container port to host port, we can provide a published port. This allows us to access the service from any node regardless of whether a container of that service (task) is running on that node.
```
docker service update --publish-add published=8080,target=80 myservice

docker-machine ls
```
Now we can access the running service from our host.

## Remove A Running Service from The Swarm
```
docker service rm myservice
```

# References

## Reading
- [Swarm mode overview](https://docs.docker.com/engine/swarm/)
- [Docker Machine](https://docs.docker.com/machine/)

## Commands
- [docker swarm](https://docs.docker.com/engine/reference/commandline/swarm/)
- [docker service](https://docs.docker.com/engine/reference/commandline/service/)
- [docker node](https://docs.docker.com/engine/reference/commandline/node/)

- [docker-machine](https://docs.docker.com/machine/reference/)