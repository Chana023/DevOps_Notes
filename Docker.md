# Docker Notes

---

### Docker Debian install

1.  Update package index
`sudo apt-get update`

2.  Install prerequisites
`sudo apt-get install apt-transport-https ca-certificates curl software-properties-common`

3.  Add Docker's official GPG key
`curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -`

4.  Set up the stable repository
`sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"`

5.  Update package index again
`sudo apt-get update`

6.  Install Docker CE
`sudo apt-get install docker-ce`

7.  Start and enable Docker service
`sudo systemctl start docker`
`sudo systemctl enable docker`

check:
`docker --version`

---

### Run image with custom name
`docker run -d -p host_port:container:port <Image name> --name CUSTOM_NAME`

`-v = volume mount`
`-e = environment variable`

---

### Delete Docker Container
`docker ps (Find Container)`

`docker stop [container name]`

`docker rm [container name]`

`docker ps -a`

---

### Copy file from host to container (this can be used vice versa)

`docker cp /path/to/local/file container_name:/path/in/container`

### Docker inspect command for checking issues on containers and accessing container logs basic

`docker inspect <Container Name>`

`docker logs -t container_name_or_id` - The -t gives timestamp