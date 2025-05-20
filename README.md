# tcpdump-docker
Docker image for tcpdump

To run this image:

```
sudo docker run -it --privileged --net=host --name=tcpdump --rm distroless-tcpdump -i any -vvnn host 127.0.0.1
```