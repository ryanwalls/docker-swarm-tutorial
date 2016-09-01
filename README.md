# docker-swarm-tutorial
In Docker 1.12, a built-in swarm mode was introduced.  This is a tutorial to walk through the basic features on a local cluster of hosts.


## Prerequisites

* Latest Docker Toolbox that supports Docker >1.12
  * Note: If you have the new native Docker applications for windows or mac, check out this link to learn how to get the native apps to play nice with Toolbox: https://docs.docker.com/docker-for-mac/docker-toolbox/#/docker-toolbox-and-docker-for-mac-coexistence.  I assume instructions are similar for Windows... but no guarantees.  If in doubt, uninstall the native apps for this tutorial.
* Internet access

## Setup machines

We are going to be following the Docker swarm "Getting Started" tutorial.  https://docs.docker.com/engine/swarm/swarm-tutorial/.  The prerequisite for the tutorial is a group of 3 machines that can communicate on ports 2377, 7946, and 4789.  Unfortunately, they don't give any guidance on setting those up... so let's do that first.

In my opinion, the easiest way to setup these machines is with `docker-machine`, which was installed when you installed Docker Toolbox.

The three machines we want to create:
* `manager1`
* `worker1`
* `worker2`

To create our 3 machines...
### On mac/linux
```
docker-machine create --driver virtualbox manager1
docker-machine create --driver virtualbox worker1
docker-machine create --driver virtualbox worker2
```

### On windows
For windows machines, if you have Hyper-V installed, you'll need to create a virtual switch and use the `hyperv` driver.  See https://github.com/SamuelDebruyn/chipsncookies-site/blob/master/content/post/run-docker-on-hyper-v-with-docker-machine.md

Once you have Hyper-V configured and your new switch configured (I named mine `docker-switch`), the commands you'll run should look like this:
```
docker-machine create -d hyperv --hyperv-virtual-switch docker-switch manager1
docker-machine create -d hyperv --hyperv-virtual-switch docker-switch worker1
docker-machine create -d hyperv --hyperv-virtual-switch docker-switch worker2
```

After you have created your machines, you'll want to set your Docker client to connect to `manager1`.  To find out how to do that run:
```
docker-machine env manager1
```

At the bottom of the output, there should be instructions on what to do next.  On Mac/Linux it should say to do the following:
```  
eval $(docker-machine env manager1)
```

## Continue Docker Tutorial

Now you can continue the tutorial from here: https://docs.docker.com/engine/swarm/swarm-tutorial/create-swarm, but with one key difference.  Whenever the tutorial asks you to ssh to a machine, use the following command to ssh to the machine:
```
docker-machine ssh <machine-name>
```

When you need the ip address of your manager machine, run the following:
```
docker-machine ip manager1
```


## Tutorial with Rancher

### Setup/Prerequisites
We are going to use Rancher to visualize what's happening on our machines.  http://rancher.com
* Start rancher on `manager1`
```
eval $(docker-machine env manager1)
docker run -d --restart=always -p 8080:8080 rancher/server
```
* Wait a few mins and then navigate to http://<manager1 ip>:8080.  
* Add all 3 machines as hosts in rancher following these directions: http://docs.rancher.com/rancher/latest/en/hosts/custom/

You are now ready to try out some various components of Docker 1.12 and the changes will be visualized in Rancher

* Create the swarm
  * Create swarm manager
```
docker swarm init --advertise-addr $(docker-machine ip manager1)
```
  * Add worker nodes to swarm.  After running the above command, the output will have commands that will need to be run on each worker node.

### Creating a network
Not new in Docker 1.12, but will need this for some future steps.

* Create an overlay network.  From Docker docs: "You can create an overlay network on a manager node running in swarm mode without an external key-value store. The swarm makes the overlay network available only to nodes in the swarm that require it for a service. When you create a service that uses the overlay network, the manager node automatically extends the overlay network to nodes that run service tasks."  See https://docs.docker.com/engine/userguide/networking/#/an-overlay-network-with-docker-engine-swarm-mode for more details.
```
docker network create --driver overlay my_network
```

### Create, scale, update, remove a service
* Create
```
docker service create --name my_web --replicas 1 --publish 8181:80 nginx:1.10
```
Verify you can hit the service from any of the machines in your cluster at port 8181.  

* Scale
```
docker service scale my_web=5
```

* Update to latest nginx
```
docker service update --image nginx:1.11 my_web
```

* Revert back to 1.10 nginx, 2 servers at a time and with a 10s delay
```
docker service update --image nginx:1.10 --update-delay 10s --update-parallelism 2 my_web
```

* Drain a node to take it out of service
```
docker node update --availability drain worker1
```

* Make the node leave the swarm
```
docker-machine ssh worker1
docker swarm leave
```

* Drain another node to take it out of service
```
docker node update --availability drain worker2
```

* Change your mind
```
docker node update --availability active worker2
```

* Remove the service
```
docker service rm my_web
```

### Try out overlay networks and routing mesh
* Create overlay network if you didn't above
```
docker network create --driver overlay my_network
```

* Create service that will be pinged
```
docker service create --network my_network --name my_web nginx
```

* Try to ping from service that is not on some network
```
docker service create --name pinger alpine ping my_web
```
What happens?

* Let's make that give up at some point
```
docker service create --name pinger --restart-max-attempts 5 alpine ping my_web
```

* Try to ping from service that is on the network
```
docker service create --name pinger --network my_network alpine ping my_web
```

* Let's confirm this with wget
Create an nginx service
```
docker service create --replicas 3 --name my_web --network my_network nginx
```
Run a box we can wget from
```
docker service create --name my-busybox --network my_network busybox sleep 30000
```
Find where busy box started:
```
docker service ps my-busybox
```
Open interactive shell in busybox
TODO
Run `nslookup` on my_web
```
nslookup my-web
```
Run wget
```
wget -O- my-web
```

* Run service in DNSRR mode instead of VIP mode
```
docker service rm my_web
```
```
docker service create --replicas 3 --name my_web --network my_network --endpoint-mode dnsrr nginx
```
Now run `nslookup` from busybox again.

### Global services
* Run something "globally" meaning one on every machine
```
docker service create --name logging --mode global alpine ping microsoft.com
```

### Labels



## Other Docker 1.12 goodness
### Health Check instruction
https://github.com/tomwillfixit/healthcheck
