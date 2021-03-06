# happner-ansible-client
Client application for managing Happn/Happner deployments. Based on [Semaphore](https://github.com/ansible-semaphore/semaphore). 

A set of sample Ansible scripts used for deploying Happner applications can be found here: https://github.com/happner/happner-ansible-orchestration



## THE DEPLOYMENT PROCESS

The process can be visualised as follows:

![ansible deployment](https://cloud.githubusercontent.com/assets/9947358/23704964/0c5560aa-0410-11e7-9ad1-4c58b9b40c4e.png)

## SEMAPHORE INSTALLATION

### Repo

- Clone the Semaphore repo from [https://github.com/ansible-semaphore/semaphore](https://github.com/ansible-semaphore/semaphore) into a local directory

- Modify the default __Dockerfile__ in the root as follows:

  ```bash
  #CMD ["/usr/bin/semaphore", "-config", "/etc/semaphore/semaphore_config.json"]
  CMD ["/bin/ash"]
  ```

- The reason for doing the above is so that you are able to access the shell (to set up the configuration file) rather than starting Semaphore straight away.

### Networking
-   In preparation for running Semaphore in a Docker container, you'll need to set up local networking to allow outgoing requests from the container to access a local MySQL database
-   set up an alias on the local loopback interface, eg: 

    ```bash
    sudo nano /etc/network/interfaces
    ...
    # modify the file as follows
    auto lo:0
    iface lo:0 inet static
    address 10.200.10.1
    ```

### MySQL

Semaphore requires a MySQL database to store deployment configurations. The database will need to provide external access to the Docker container - see below:

#### Installation of MySQL on Ubuntu

-   A great resource for setting MySQL up on Ubuntu can be found here: [http://www.configserverfirewall.com/ubuntu-linux/enable-mysql-remote-access-ubuntu/](http://www.configserverfirewall.com/ubuntu-linux/enable-mysql-remote-access-ubuntu/)

-   Check the bindings - modify `/etc/mysql/mysql.conf.d/mysqld.cnf` as follows:

    ```bash
    bind-address = 0.0.0.0	#this allows all - use 10.200.10.1 to be more restrictive  
    ```

    Then restart the service 

    ```
    sudo systemctl restart mysql.service
    ```

#### Installation of MySQL on OSX

- https://coderwall.com/p/os6woq/uninstall-all-those-broken-versions-of-mysql-and-re-install-it-with-brew-on-mac-mavericks

-   Check the bindings on the local MySQL using: 
          `ps -ax | grep mysql`
    -   this will show a list of processes for MySQL - you should see a line that reads 

            `bind-address=...`
    -   if this is __127.0.0.1__ you'll need to change the value to the alias created above

-   Locate the relevant file to change the bindings:
    - if you've used __brew__ to install MySQL, it should be in `usr/local/Cellar/mysql/[version]/homebrew.mxcl.mysql.plist`

-   Modify the file: 
    `nano /usr/local/Cellar/mysql/5.7.17/homebrew.mxcl.mysql.plist`

    - change the line that says `<string>--bind-address=127.0.0.1</string>` to your alias address or `*` or `0.0.0.0`

-   Restart MySQL:
    - `brew services restart mysql`

-   Test the remote connection to local MySQL using:
    - `mysql -h 10.200.10.1 -u root -p`

#### Create the database and user

- `CREATE USER 'semaphore_user'@'10.200.10.1' IDENTIFIED BY 'password';` 
  OR (to allow connections from anywhere)
  `CREATE USER 'semaphore_user'@'%' IDENTIFIED BY 'password';`
- NOTE: on an EC2 instance this will be something like:
  ` create user 'semaphore_user'@'ip-10-200-10-1.eu-west-1.compute.internal' identified by 'password';`
- `CREATE DATABASE semaphore;`
- `GRANT ALL PRIVILEGES ON semaphore.* TO 'semaphore_user'@'10.200.10.1';`
- `FLUSH PRIVILEGES;`

### Docker

If you don't already have Docker installed on the deployment server, follow the instructions below for installation:

- OSX:
  - Use the native Docker installer for OSX rather than the Docker Toolbox:
    - [https://www.docker.com/products/docker#/mac](https://www.docker.com/products/docker#/mac)
- Ubuntu:
  - [https://www.digitalocean.com/community/tutorials/how-to-install-and-use-docker-on-ubuntu-16-04](https://www.digitalocean.com/community/tutorials/how-to-install-and-use-docker-on-ubuntu-16-04)

### Docker image
- From the root of the cloned project, you can now build the Docker image using:

  `sudo docker build -t happner/ansible-client:v1 .`
- Once its built, you will be able to start a container as follows:

```bash
docker run -e SEMAPHORE_DB=semaphore -e SEMAPHORE_DB_HOST=10.200.10.1 -e SEMAPHORE_DB_PORT=3306 -e SEMAPHORE_DB_USER=semaphore_user -e SEMAPHORE_DB_PASS=password 
-e SEMAPHORE_ADMIN=admin -e SEMAPHORE_ADMIN_NAME=admin -e SEMAPHORE_ADMIN_EMAIL=admin@test.com -e SEMAPHORE_ADMIN_PASSWORD=password -e  ANSIBLE_HOST_KEY_CHECKING=false -p 3000:3000 -it --rm happner/ansible-client:v1
```

-  The above ENV variables (signified by an __-e__ switch) will ensure that Semaphore can connect to the local MySQL instance and will also create a default __admin__ user

-  Sample output of starting the container:

   ```bash
   > Username:  > Email:  > Your name:  > Password: 
   You are all setup !
   Re-launch this program pointing to the configuration file
   ....
   ```

-  Once the container is started, you can start Semaphore:

   ```bash
   /usr/bin/semaphore -config /tmp/semaphore/semaphore_config.json
   ```

-  The API endpoints should all be displayed in the container terminal

-  You'll now be able to access the web application via __localhost:3000__!

   - __*__ the admin login credentials are the ENV vars that you passed in when starting the container (see docker run above):  __admin__ : __password__

-  Good luck ;-)


## CREATING DEPLOYMENT SCENARIOS IN SEMAPHORE

Once you have Semaphore up and running, you can start to create deployment scenarios based on Ansible scripts that you have created. See https://github.com/happner/happner-ansible-orchestration for samples.

### Terms used

- __Repositories__
  - these are the Github repos that contain the Ansible playbook/roles required for a deployment
  - any SSH keys required to access the repo are referred to here (see below)
- __Key Store__
  - this is where the SSH keys are saved for access to:
    - Github repos
    - deployment targets (servers that playbooks will be deploying to) - i.e.: __Inventory__
- __Environment__
  - a label for a group of servers
- __Inventory__
  - this is where the individual servers are listed for an environment
  - each server is a deployment target
- __Task templates__
  - this is where all the above variables can be assembled to create a deployment scenario



### Walkthrough - creating a deployment scenario

Assuming that you have already followed these simple steps in the relevant tabs:

- __Key Store__ - added SSH keys for Github repo and for access to deployment server/s
- __Playbook Repository__ - added your Ansible playbook repo URI and associated SSH key
- __Environment__ - created an environment label

The remaining steps (in detail) are:

- __Inventory__ :

  in the Inventory tab, click the "create inventory" button to produce the following modal. Enter your selected details:

  ![inventory_semaphore_0](https://cloud.githubusercontent.com/assets/9947358/23649210/542b5596-0326-11e7-90f6-b7be64c6a4c3.png)
  ​

  once you've saved this, click the "edit inventory content" button on this line item:

  ![inventory_semaphore_1](https://cloud.githubusercontent.com/assets/9947358/23649211/543426f8-0326-11e7-89eb-229bc411557d.png)

  fill in your server IP addresses (and optionally host name and user), one server per line

  ![inventory_semaphore_2](https://cloud.githubusercontent.com/assets/9947358/23649212/543c4a54-0326-11e7-9754-a636da5e119d.png)
  ​



- __Task Templates__ :

  click the "new template" button to create a new task template:

  ![templates_semaphore_0](https://cloud.githubusercontent.com/assets/9947358/23649214/54448e4e-0326-11e7-8899-20a69ee4912e.png)
  ​

  save this...
  ​
  ![templates_semaphore_1](https://cloud.githubusercontent.com/assets/9947358/23649213/54438314-0326-11e7-8be2-63b72d3c1022.png)

  You're now ready to run the task - press the "run" button (you can select debug for a detailed trace of the Ansible output). The overrides are optional.

  ![templates_semaphore_2](https://cloud.githubusercontent.com/assets/9947358/23649215/5447f94e-0326-11e7-993e-a8e913565d29.png)

  ​

  If you have a successful run, you'll see output something like this:

  ![templates_semaphore_3](https://cloud.githubusercontent.com/assets/9947358/23649216/546d7124-0326-11e7-8351-69533c80d2cc.png)
  ​

- You should now have successfully deployed to your servers!