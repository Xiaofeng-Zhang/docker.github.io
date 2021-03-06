---
description: Deploying Apps on Docker for Azure
keywords: azure, microsoft, iaas, deploy
title: Deploy your app on Docker for Azure
---

## Connecting to your manager nodes using SSH

This section will walk you through connecting to your installation and deploying
applications.

First, you will obtain the public IP address for a manager node. Any manager
node can be used for administrating the swarm.

##### Manager Public IP and SSH ports on Azure

Once you've deployed Docker on Azure, go to the "Outputs" section of the
resource group deployment.

![SSH targets](img/sshtargets.png)

The "SSH Targets" output is a URL to a blade that describes the IP address
(common across all the manager nodes) and the SSH port (unique for each manager
node) that you can use to log in to each manager node.

![Swarm managers](img/managers.png)

Obtain the public IP and/or port for the manager node as instructed above and
use the provided SSH key to begin administrating your swarm and the unique port associated with a manager:

    $ ssh -i <path-to-ssh-key> -p <ssh-port> docker@<ip>
    Welcome to Docker!

Once you are logged into the container you can run Docker commands on the swarm:

    $ docker info
    $ docker node ls

You can also tunnel the Docker socket over SSH to remotely run commands on the cluster (requires [OpenSSH 6.7](https://lwn.net/Articles/609321/) or later):

    $ ssh -i <path-to-ssh-key> -p <ssh-port> -fNL localhost:2374:/var/run/docker.sock docker@<ssh-host>
    $ docker -H localhost:2374 info

If you don't want to pass `-H` when using the tunnel, you can set the `DOCKER_HOST` environment variable to point to the localhost tunnel opening.

## Connecting to your Linux worker nodes using SSH

The Linux worker nodes have SSH enabled. SSH access is not possible to the worker nodes from the public
Internet directly. To access the worker nodes, you will need to first connect to a
manager node (see above) and then `ssh` to the worker node, over the private
network. Make sure you have SSH agent forwarding enabled (see below). If you run
the `docker node ls` command you can see the full list of nodes in your swarm.
You can then `ssh docker@<worker-host>` to get access to that node.

##### Configuring SSH agent forwarding

SSH agent forwarding allows you to forward along your ssh keys when connecting from one node to another. This eliminates the need for installing your private key on all nodes you might want to connect from.

You can use this feature to SSH into worker nodes from a manager node without
installing keys directly on the manager.

If your haven't added your ssh key to the `ssh-agent` you will also need to do this first.

To see the keys in the agent already, run:

```
$ ssh-add -L
```

If you don't see your key, add it like this.

```
$ ssh-add ~/.ssh/your_key
```

On Mac OS X, the `ssh-agent` will forget this key, once it gets restarted. But you can import your SSH key into your Keychain like this. This will have your key survive restarts.

```
$ ssh-add -K ~/.ssh/your_key
```

You can then enable SSH forwarding per-session using the `-A` flag for the ssh command.

Connecting to the Manager.
```
$ ssh -A docker@<manager ip>
```

To always have it turned on for a given host, you can edit your ssh config file
(`/etc/ssh_config`, `~/.ssh/config`, etc) to add the `ForwardAgent yes` option.

Example configuration:

```
Host manager0
  HostName <manager ip>
  ForwardAgent yes
```

To SSH in to the manager with the above settings:

```
$ ssh docker@manager0
```

## Connecting to your Windows worker nodes using RDP

The Windows worker nodes have RDP enabled. RDP access is not possible to the worker nodes from the public
Internet. To access the worker nodes using RDP, you will need to first connect to a
manager node (see above) over `ssh`, establish a SSH tunnel and then use RDP to connect to the worker node over the SSH tunnel.

To get started, first login to a manager node and determine the private IP address of the Windows worker VM:

```
$ docker node inspect <windows-worker-node-id> | jq -r ".[0].Status.Addr"
```

Next, in your local machine, establish the SSH tunnel to the manager for the RDP connection to the worker. The local port can be any free port and typically a high value e.g. 9001.

```
$ ssh -L <local-port>:<windows-worker-ip>:3389 -p <manager-ssh-port> docker@<manager-ip>
```

Finally you can use a RDP client on your local machine to connect to `local-host`:`local-port` and the connection will be forwarded to the RDP server running in the Windows worker node over the SSH tunnel created above.

## Running apps

You can now start creating containers and services.

    $ docker run hello-world

You can run websites too. Ports exposed with `--publish` are automatically exposed through the platform load balancer:

    $ docker service create --name nginx --publish target=80,port=80 nginx

Once up, find the `DefaultDNSTarget` output in either the AWS or Azure portals to access the site.

### Execute docker commands in all swarm nodes

There are cases (such as installing a volume plugin) wherein a docker command may need to be executed in all the nodes across the cluster. You can use the `swarm-exec` tool to achieve that.

Usage : `swarm-exec {Docker command}`

The following will install a test plugin in all the nodes in the cluster.

Example : `swarm-exec docker plugin install --grant-all-permissions mavenugo/test-docker-netplugin`

This tool internally makes use of docker global-mode service that runs a task on each of the nodes in the cluster. This task in turn executes your docker command. The global-mode service also guarantees that when a new node is added to the cluster or during upgrades, a new task is executed on that node and hence the docker command will be automatically executed.

### Docker Stack deployment

To deploy complex multi-container apps, you can use the `docker stack deploy` command. You can either deploy a bundle on your machine over an SSH tunnel, or copy the `docker-compose.yml` file (for example using `scp`) to a manager node, SSH into the manager and then run `docker stack deploy` (if you have multiple managers, you have to ensure that your session is on one that has the stack file).

For example:

```bash
docker stack deploy -f docker-compose.yml myapp
```

A good sample app to test deployment of stacks is the [Docker voting app](https://github.com/docker/example-voting-app).

By default, apps deployed with stacks do not have ports publicly exposed. Update port mappings for services, and Docker will automatically wire up the underlying platform load balancers:

    docker service update --publish-add target=80,port=80 <example-service>

### Images in private repos

To create swarm services using images in private repos, first make sure you're authenticated and have access to the private repo, then create the service with the `--with-registry-auth` flag (the example below assumes you're using Docker Hub):

    docker login
    ...
    docker service create --with-registry-auth user/private-repo
    ...

This will cause swarm to cache and use the cached registry credentials when creating containers for the service.
