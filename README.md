# tcpdump-docker: A minimal tcpdump Docker Image

This project offers a Docker image for the `tcpdump` command-line utility, built on a distroless Debian base. It provides a minimal environment containing the essential `tcpdump` binary and its dependencies for capturing network traffic.

## Usage

```
sudo docker run -it --privileged --net=host --name=tcpdump --rm ghcr.io/kubebee-com/tcpdump -i any -vvnn host 127.0.0.1
```