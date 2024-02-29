This repository contains the files/instructions on reproducing an issue that we've seen on EKS IPv6 clusters using AWS VPC CNI.

Reproduced across AL2/Ubuntu/Bottlerocket with Kernel versions:
- AL2: 5.10.209-198.858.amzn2.aarch64 / 5.10.209-198.858.amzn2.x86_64
- Ubuntu 22: 6.2.0-1017-aws #17~22.04.1-Ubuntu SMP
- Ubuntu 20: 5.15.0-1048-aws #53~20.04.1-Ubuntu SMP
- Bottlerocket: 1.18.0-7452c37e , 1.19.2-29cc92cc

Tested on AWS VPC CNI versions:
- v1.16.3-eksbuild.2
- v1.15.1-eksbuild.1

Instance types tested:
- m6g.xlarge
- c6g.xlarge
- m7a.8xlarge
- m6a.8xlarge

### Reproduction Steps

It may take a few attempts to reproduce errors, but the observed behavior is that large simultaneous downloads stall out and eventually we receive a "connection reset by peer" error.

#### Debian Slim Container

Launch a container:  `kubectl run -it --rm ipv6-reset-test-debian --image public.ecr.aws/debian/debian:bullseye-slim --command -- bash`

```
apt-get update && apt-get install -y wget
ARCH="$(arch | sed s/aarch64/arm64/ | sed s/x86_64/amd64/)"
wget https://github.com/wistia/hivemind/releases/download/v1.1.1/hivemind-v1.1.1-wistia-linux-$ARCH.gz
gunzip hivemind-v1.1.1-wistia-linux-$ARCH.gz
mv hivemind-v1.1.1-wistia-linux-$ARCH hivemind
chmod +x hivemind
wget https://raw.githubusercontent.com/wistia/eks-ipv6-reset-example/main/Procfile
./hivemind -W
```

#### Alpine Container

Launch a container: `kubectl run -it --rm ipv6-reset-test-debian --image public.ecr.aws/docker/library/alpine:3.19.1 --command -- ash`
`

```
apk add wget # use non-busybox wget
ARCH="$(arch | sed s/aarch64/arm64/ | sed s/x86_64/amd64/)"
wget https://github.com/wistia/hivemind/releases/download/v1.1.1/hivemind-v1.1.1-wistia-linux-$ARCH.gz
gunzip hivemind-v1.1.1-wistia-linux-$ARCH.gz
mv hivemind-v1.1.1-wistia-linux-$ARCH hivemind
chmod +x hivemind
wget https://raw.githubusercontent.com/wistia/eks-ipv6-reset-example/main/Procfile
./hivemind -W
```


#### Example error output

```
  2  723M    2 17.2M    0     0  18871      0 11:09:59  0:16:00 10:53:59     0* Recv failure: Operation timed out
test2 | * OpenSSL SSL_read: Operation timed out, errno 110
test2 | * Failed receiving HTTP2 data: 56(Failure when receiving data from the peer)
  2  723M    2 17.2M    0     0  18866      0 11:10:09  0:16:01 10:54:08     0
test2 | * Connection #0 to host embed-ssl.wistia.com left intact
test2 | curl: (56) Recv failure: Operation timed out
test2 | exit status 56
 38  723M   38  280M    0     0   291k      0  0:42:22  0:16:25  0:25:57     0* Recv failure: Operation timed out
test8 | * OpenSSL SSL_read: Operation timed out, errno 110
test8 | * Failed receiving HTTP2 data: 56(Failure when receiving data from the peer)
 38  723M   38  280M    0     0   291k      0  0:42:25  0:16:26  0:25:59     0
test8 | * Connection #0 to host embed-ssl.wistia.com left intact
test8 | curl: (56) Recv failure: Operation timed out
test8 | exit status 56
```
