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

To create our 3 machines, run each of the following:
```
docker-machine create --driver virtualbox manager1
docker-machine create --driver virtualbox worker1
docker-machine create --driver virtualbox worker2
```

After you have created your machines, you'll want to set your Docker client to connect to `manager1`.  To find out how to do that run:
```
docker-machine env manager1
```

At the bottom of the output, there should be instructions on what to do next.  On Mac/Linux it should say to do the following:
```  
eval $(docker-machine env manager1)
```

Now you can continue the tutorial from here: https://docs.docker.com/engine/swarm/swarm-tutorial/create-swarm, but with one key difference.  Whenever the tutorial asks you to ssh to a machine, use the following command to ssh to the machine:
```
docker-machine ssh <machine-name>
```
