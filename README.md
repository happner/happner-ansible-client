# happner-ansible-client
Client application for happner-ansible-orchestration based on Semaphore

## Developer notes

As this is based on Semaphore, an open-source client, the provided Dockerfile was used to get things running.

### Networking
- In preparation for running Semaphore in a Docker container, you'll need to set up local networking to allow outgoing
       requests from the container to access a local MySQL database:
    - set up an alias on the local loopback interface, eg: 
      `sudo ifconfig lo0 alias 10.200.10.1/24`
    - to remove this just use: 
      `sudo ifconfig lo0 -alias 10.200.10.1`

### External access to a local MySQL database
- Your local database will need to provide external access to the Docker container
- Check the bindings on the local MySQL using: 
      `ps -ax | grep mysql`
    - this will show a list of processes for MySQL
    - you should see a line that reads 
        `bind-address=...`
    - if this is __127.0.0.1__ you'll need to change the value to the alias created above
    - locate the relevant file to change this value (you should find this in your grep result):
        - if you've used __brew__ to install MySQL, it should be in `usr/local/Cellar/mysql/[version]/homebrew.mxcl.mysql.plist`
        - `nano /usr/local/Cellar/mysql/5.7.17/homebrew.mxcl.mysql.plist`
        - change the line that says 
          `<string>--bind-address=127.0.0.1</string>`
           to your alias address or `*` or `0.0.0.0`
        - __BE CAREFUL, IF YOU USE A WILDCARD THIS GIVES ACCESS TO ANYBODY__
    - restart MySQL:
        - `brew services restart mysql`

### Docker on OSX
- Use the native Docker installer for OSX rather than the Docker Toolbox:
  - [https://www.docker.com/products/docker#/mac](https://www.docker.com/products/docker#/mac)

### Docker image
- Build the image using something like:

  `sudo docker build -t happner/ansible-client:v1 .`
- Once its built, start a container as follows:

```bash
docker run --add-host=mysql_host:10.200.10.1 -e SEMAPHORE_DB=semaphore -e SEMAPHORE_DB_HOST=mysql_host -e  SEMAPHORE_DB_USER=semaphore_user -e SEMAPHORE_DB_PASS=password -it --rm happner/ansible-client:v1`
```
