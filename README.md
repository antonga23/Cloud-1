# Deploying WordPress on Elastic Beanstalk
Use the EB CLI to create an Elastic Beanstalk environment with an attached RDS DB and EFS file system to provide WordPress with a MySQL database and shared storage for uploaded files.

## Install the EB CLI

The EB CLI integrates with Git and simplifies the process of creating environments, deploying code changes, and connecting to the instances in your environment with SSH. You will perform all of these activites when installing and configuring WordPress.

If you have pip, use it to install the EB CLI.

```Shell
$ pip install awsebcli
```

Add the local install location to your OS's path variable. The installation path depends on where you installed Python. Note that it is recommended to install Python in your user directory, and avoid using the version of Python that came with your operating system.

###### Linux
```Shell
$ export PATH=~/.local/bin:$PATH
```
###### OS-X
```Shell
$ export PATH=~/Library/Python/3.6/bin:$PATH
```
###### Windows
Add `%USERPROFILE%\AppData\Roaming\Python\Scripts` to your PATH variable. Search for **Edit environment variables for your account** in the Start menu.

If you don't have pip, follow the instructions [here](http://docs.aws.amazon.com/elasticbeanstalk/latest/dg/eb-cli3-install.html).

## Set up your project directory

1. Download WordPress.

        ~$ curl https://wordpress.org/wordpress-4.8.3.tar.gz -o wordpress.tar.gz

2. Download the configuration files in this repository.

        ~$ wget https://github.com/awslabs/eb-php-wordpress/releases/download/v1.0/eb-php-wordpress-v1.zip

3. Extract WordPress and change the name of the folder.

        ~$ tar -xvf wordpress.tar.gz
        ~$ mv wordpress wordpress-beanstalk
        ~$ cd wordpress-beanstalk

4. Extract the configuration files over the WordPress installation.

        ~/wordpress-beanstalk$ unzip ../eb-php-wordpress-v1.zip
         creating: .ebextensions/
        inflating: .ebextensions/dev.config
        inflating: .ebextensions/efs-create.config
        inflating: .ebextensions/efs-mount.config
        inflating: .ebextensions/loadbalancer-sg.config
        inflating: .ebextensions/wordpress.config
        inflating: LICENSE
        inflating: README.md
        inflating: wp-config.php

## Create an Elastic Beanstalk environment

1. Configure a local EB CLI repository with the PHP platform. Choose a [supported region](http://docs.aws.amazon.com/general/latest/gr/rande.html#elasticbeanstalk_region) that is close to you.

        ~/wordpress-beanstalk$ eb init --platform php7.0 --region us-west-2
        Application wordpress-beanstalk has been created.

2. Configure SSH. Create a key that Elastic Beanstalk will assign to the EC2 instances in your environment to allow you to connect to them later. You can also choose an existing key pair if you have the private key locally.

        ~/wordpress-beanstalk$ eb init
        Do you want to set up SSH for your instances?
        (y/n): y

        Select a keypair.
        1) [ Create new KeyPair ]
        (default is 1): 1

        Type a keypair name.
        (Default is aws-eb): beanstalk-wordpress

3. Create an Elastic Beanstalk environment with a MySQL database.

        ~/wordpress-beanstalk$ eb create wordpress-beanstalk --sample --database
        Enter an RDS DB username (default is "ebroot"):
        Enter an RDS DB master password:
        Retype password to confirm:
        Environment details for: wordpress-beanstalk
          Application name: wordpress-beanstalk
          Region: us-west-2
          Deployed Version: Sample Application
          Environment ID: e-nrx24yzgmw
          Platform: 64bit Amazon Linux 2016.09 v2.2.0 running PHP 7.0
          Tier: WebServer-Standard
          CNAME: UNKNOWN
          Updated: 2016-11-01 12:20:27.730000+00:00
        Printing Status:
        INFO: createEnvironment is starting.

## Networking configuration
Modify the configuration files in the .ebextensions folder with the IDs of your [default VPC and subnets](https://console.aws.amazon.com/vpc/home#subnets:filter=default), and [your public IP address](https://www.google.com/search?q=what+is+my+ip).

 - `.ebextensions/dev.config` restricts access to your environment to your IP address to protect it during the WordPress installation process. Replace the placeholder IP address near the top of the file with your public IP address.
 - `.ebextensions/efs-create.config` creates an EFS file system and mount points in each Availability Zone / subnet in your VPC. Identify your default VPC and subnet IDs in the [VPC console](https://console.aws.amazon.com/vpc/home#subnets:filter=default). If you have not used the console before, use the region selector to select the same region that you chose for your environment.


## Deploy WordPress to your environment
Deploy the project code to your Elastic Beanstalk environment.

```Shell
~/wordpress-beanstalk$ eb status
~/wordpress-beanstalk$ eb deploy
```

## Security 

This project includes a configuration file (`loadbalancer-sg.config`) that creates a security group and assigns it to the environment's load balancer, using the IP address that you configured in `ssh.config` to restrict HTTP access on port 80 to connections from your network.

## Install WordPress

Open the site in a browser.

```Shell
~/wordpress-beanstalk$ eb open
```


Perform a standard installation. The `wp-config.php` file is already present in the source code and configured to read database connection information from the environment.

Installation takes about a minute to complete.

# Updating keys and salts

The WordPress configuration file `wp-config.php` also reads values for keys and salts from environment properties.

The hash salt can be any value but shouldn't be stored in source control. Use `eb setenv` to set these properties directly on the environment.

    AUTH_KEY, SECURE_AUTH_KEY, LOGGED_IN_KEY, NONCE_KEY, AUTH_SALT, SECURE_AUTH_SALT, NONCE_SALT

```Shell
~/wordpress-beanstalk$ eb setenv AUTH_KEY=29dl39gksao SECURE_AUTH_KEY=ah24h3drfh LOGGED_IN_KEY=xmf7v0k27d5fj3 ...
...
```

Setting the properties on the environment directly by using the EB CLI or console overrides the values in `wordpress.config`.

Remove the custom load balancer configuration to open the site to the Internet.

```Shell
~/wordpress-beanstalk$ rm .ebextensions/loadbalancer-sg.config
~/wordpress-beanstalk$ eb deploy
```

Scale up to run the site on multiple instances for high availability.
```Shell
~/wordpress-beanstalk$ eb scale 3
```

When the update completes, open the site.

```Shell
~/wordpress-beanstalk$ eb open
```

Refresh the site several times to verify that all instances are reading from the EFS file system. Create posts and upload files to confirm functionality.
