Ansible playbooks to deploy Docker Swarm on a Raspberry Pi Cluster
-----------------------------------------------------------------------------

- Requires six Raspberry Pi devices (v3 was used in the demo, v2 also works fine)
- Requires six micro-SD cards (8GB)
- Requires Ansible (2.2 was used in the demo)
- Requires [Hypriot flash tool](https://github.com/hypriot/flash)
- Requires [HypriotOS](https://blog.hypriot.com/downloads/) (1.1.3 was used in the demo)
- Requires HypriotOS installed on the micro-SD cards
- Requires hostnames configured on the micro-SD cards matching the names in the inventory file

- Expects the six Raspberry Pi devices powered on and connected on the same network as the machine running Ansible
- Expects an RGB Common Cathode led connected to the GPIO pins. Red connected to pin 15 (GPIO22), green connected to pin 13 (GPIO13), blue connected to pin 11 (GPIO17) and the ground connected to pin 9. If you have a Common Anode RGB led or a different pin setup change the values for ledOn and ledOff or GPIO pins in the vars.yml file.

Note: If you do not have the leds installed you will miss out on the visuals but the playbooks should work fine.

These playbooks have been demonstrated at the "DockerGrunn Meetup #7 - Winter is coming" event on 18 January 2017. The slide deck is available at [Speaker Deck](https://speakerdeck.com/tisgoud/dockergrunn-meetup-number-7-docker-swarm-on-a-raspberry-pi-cluster).

### Preparing the micro-SD cards

Download [HypriotOS](https://blog.hypriot.com/downloads/) and instal the [Hypriot flash tool](https://github.com/hypriot/flash).
Install HypriotOS with the Hypriot flash tool on the micro-SD cards:

 		flash -hostname [name] hypriotos-rpi-v1.1.3.img.zip

Use the name(s) of the swarmManager and swarmNodes from the inventory file


### Set up SSH

If you use homebrew then follow the steps below otherwise follow the instructions on [How to set up SSH Keys](https://www.digitalocean.com/community/tutorials/how-to-set-up-ssh-keys--2)

Install ssh-copy-id with brew:

		brew install ssh-copy-id

Repeat for each node:

		ssh-copy-id -i ~/.ssh/id_rsa.pub pirate@[name]

Use the name(s) of the swarmManager and swarmNodes from the inventory file

### Initial Swarm Setup

In the demo the following inventory file is used:

		# Inventory file for a six-node Raspberry Pi cluster

		[swarmManagers]
		blacknode.local

		# Put additional Swarm Managers here (not a part of the demo, use at own risk)
		[additionalSwarmManagers]

		[swarmNodes]
		greennode.local
		rednode.local
		pinknode.local
		yellownode.local
		whitenode.local

		[swarm:children]
		swarmManagers
		swarmNodes

		[swarm:vars]
		ansible_ssh_user=pirate

To set up the GPIO on the Raspberry Pi devices we run the following playbook:

		ansible-playbook -i inventory gpioSetup.yml

To deploy the swarm cluster you execute the following command:

		ansible-playbook -i inventory swarmSetup.yml

The first time this playbook is executed it takes a couple of minutes. A docker image is downloaded and this takes some time.

The following steps will be executed:

- The visualization container is started
- The swarm is initialized
- On the machine running Ansible the visualization page is opened
- The nodes are added to the cluster

### Running some services

With the cluster up and running we can start some services:

		ansible-playbook -i inventory servicesSetup.yml

This will start the following services:

- busybox service
- whoami service

Additionally the web page of both services is opened in the default browser

### Playing around with Docker Swarm

Initially only one instance of each service is running. The next step is to add more instances of the services:

		docker service update whoami --replicas 10
		docker service update busybox --replicas 10

Check the visualization page to see the additional instances running.

In the following step we will remove the "white" node from the cluster:

		ansible-playbook -i inventory whitenodeTeardown.yml

When we look at the visualization page we see that the number of instances is still the same.

Re-add the "white" node:

		ansible-playbook -i inventory whitenodeSetup.yml

Notice that we now can see the "white" node in the visualization page but there are no service instances running on that node.

When we update the services to add more instances the load will be distributed across al nodes:

		docker service update whoami --replicas 18
		docker service update busybox --replicas 18

Again check the visualization page to see that the "white" node is running service instances.

### Teardown sequence

To clean up the environment we execute the setup steps in the reverse order.

Shutdown the services:

		ansible-playbook -i inventory servicesTeardown.yml

Teardown the cluster:

		ansible-playbook -i inventory swarmTeardown.yml

Remove the GPIO files (optional):

		ansible-playbook -i inventory gpioTeardown.yml

Shutdown the Raspberry Pi stack:

		ansible-playbook -i inventory shutdown.yml

The only remaining manual step in cleaning up is to close the three browser tabs.
