# Creating the Spinnaker VPC

## Several things to Note:
* Store your terraform state file and it's backup in a secure place. It's not a good idea to push it to a public repository.
* The script is designed to be run on the same host as where you would be creating the SSH tunnel and browsing spinnaker from.
* Only supports AWS right now.

## To use:
* Install Terraform (https://terraform.io/downloads.html) and make sure it's in your $PATH
* Set your AWS ENV Variables.
* Clone this repo
* checkout 'spinnaker' branch
* generate ssh key. This should not be your default ssh key.
* Look at ./aws/terraform.tfvars and change anything you think might need changing (region, vpc_name, vpc_cidr)
  * set ssh_private_key_location with the filesystem location of the ssh private key you created.
  * set ssh_public_key to be the value of the public key.
  * set adm_bastion_incoming_cidrs and infra_jenkins_incoming_cidrs to a comma separated list of CIDRS that need to access those services. 22 is open to these IPs on the bastion host, and 80 is open to these IPs on the jenkins host.
  * for now, do not change ssh_key_name
* run the script:
```
./create_spinnaker_vpc.sh -a apply -c aws
```
There are two optional flags you can pass to the create_spinnaker_vpc.sh script
```
-l Tells the script to log the terraform output to a file. Location of file will be printed.
-i <path where to store tf state files>. If you don't want the tfstate files to be stored in the default location ('./')
```

You can also put 'plan' in place of 'apply', and 'terraform plan' will be run, which will show you what terraform would do if you ran apply. It's a good way to test changes to any of the .tf files as most syntax errors will be caught.

... wait 15 minutes or so ...
Pay careful attention to the output at the end, example:
```
Outputs:

   =

Jenkins Public IP (for DNS): "52.34.56.108"
Bastion Public IP (for DNS): "52.32.185.147"

Configure known hosts on the bastion server:
	--- cut ---
	ssh -o IdentitiesOnly=yes -i /Users/username/.ssh/id_rsa_spinnaker_terraform ubuntu@52.32.185.147 'ssh-keyscan -H 192.168.3.189 > ~/.ssh/known_hosts'
	--- end cut ---
NOTE: THIS needs to be done BEFORE you can run the following tunnel command.

Now, start up a tunnel:

SSH CLI command:
	--- cut ---
	ssh -o IdentitiesOnly=yes -i /Users/username/.ssh/id_rsa_spinnaker_terraform -L 8080:localhost:8080 -L 8084:localhost:8084 ubuntu@52.32.185.147 'ssh -o IdentitiesOnly=yes -i /home/ubuntu/.ssh/id_rsa -L 8080:localhost:80 -L 8084:localhost:8084 -A ubuntu@192.168.3.189'
	--- end cut ---

To create an example pipeline:
    --- cut ---
    cd support ; ./create_application_and_pipeline.py -a appname -p appnamepipeline -g sg-30165d54 -v sg-31165d55 -m sg-3c165d58
    --- end cut ---
```

To create the tunnel, you need to do two things (from the example output above)
* The following sets up ssh keys nicely on the bastion host:
```
ssh -o IdentitiesOnly=yes -i /Users/username/.ssh/id_rsa_spinnaker_terraform ubuntu@52.32.185.147 'ssh-keyscan -H 192.168.3.189 > ~/.ssh/known_hosts'
```
* The following creates the actual tunnel:
```
ssh -o IdentitiesOnly=yes -i /Users/username/.ssh/id_rsa_spinnaker_terraform -L 8080:localhost:8080 -L 8084:localhost:8084 ubuntu@52.32.185.147 'ssh -o IdentitiesOnly=yes -i /home/ubuntu/.ssh/id_rsa -L 8080:localhost:80 -L 8084:localhost:8084 -A ubuntu@192.168.3.189'
```

* With the tunnel running you can go to http://localhost:8080/ to access Spinnaker.
* NOTE: Jenkins does NOT have to be accessed through the tunnel, and you should be able to login to it using the public IP (if you set infra_jenkins_incoming_cidrs correctly) with the username/password in terraform.tfvars

## Creating a pipeline for the example app:
You'll see a line in the example output above that looks like this:
```
cd support ; ./create_application_and_pipeline.py -a appname -p appnamepipeline -g sg-30165d54 -v sg-31165d55 -m sg-3c165d58
```
Execute it, and it will create a pipeline in Spinnaker. This requires that your AWS ENV vars be set.

With a working pipeline, all you should have to do is go to the 'Package_example_app' job on jenkins and build it. The Spinnaker pipeline will be trigged, an AMI baked, and an ASG deployed with a Load Balancer.

# Destroying the spinnaker VPC
Before running terraform destroy, you need to execute several manual steps to destroy the VPC that was created
* Terminate any instances that were created by spinnaker.
* Delete any ASGs that were created by spinnaker
* Delete any Load Balancers that were created by spinnaker
* Delete any Launch Configurations that were created by spinnaker

If you do not do the previous steps terraform will not be able to completely destroy the VPC.

Run this command:
```
./create_spinnaker_vpc.sh -a destroy -c aws
```
Congratulations, your spinnaker VPC is now gone!