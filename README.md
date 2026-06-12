# Trial slurm installation in docker

This is a repo for creating a mock [slurm](https://slurm.schedmd.com/documentation.html) installation inside a docker cluster
The idea is to launch the containers as an uninitialized machine and than deploy slurm on them using ansible (following the [Infrastructure as Code (IaC)](https://en.wikipedia.org/wiki/Infrastructure_as_code) paradigm).
After that you can go inside the containers and play with all the configurations or follow the examples.

# Andrew's Updates

I updated the compose.yml to make use of cgroups v2 so the below command will now execute correctly. 

```
docker compose up --build -d --remove-orphans
```

I was called by a buddy with questions about SLURM and I thought I would aquaint myself with it a bit more. I asked Claude for a quick run down and some break/fix options to test and see outcomes. Those files are:

- [slurm-cheatsheet.md](slurm-cheatsheet.md)
- [slurm-break-it-drills.md](slurm-break-it-drills.md)

As of right now everything spins up and works. If you don't have ansible installed you can execute the ansible-palybook command by running my [docker-devops container](https://github.com/andrewjkrull/docker-devops) with the following command:

```bash
docker run --rm -it --network slurm-network \
  -v "$PWD/ansible:/work" -w /work \
  devops-toolkit:latest \
  sh -c 'ansible-galaxy collection install community.mysql && ansible-playbook slurm.yml -i test_hosts.yml'
```

In the future I might take some time to update the compose file to spin up a MySQL container instead of installing one on the control node. 

## Usage

Run the following command to bring up the docker containers (`--remove-orphans` is used to remove services that are no longer recognized after modifying the compose file):

    docker compose up --build -d --remove-orphans

In order to start from the beginning (e.g. for a new example) you can remove the created containers by running:

    docker compose rm -fs

If not already available, [install ansible](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html)

Go inside the ansible directory and run the ansible script to setup slurm in the docker cluster:

    ansible-playbook slurm.yml -i test_hosts.yml

Now you should be able to either ssh into your control node as root with password `root`:

    ssh root@172.19.0.2

Or spawn a shell inside of it (change container name id needed):

    docker exec -it docker-slurm-tutorial-control /bin/bash

## Examples

- [Run a simple job and modify a QoS](example1/README.md)
- [Add new nodes to an existing partition](example2/README.md)

