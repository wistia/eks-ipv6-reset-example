This repository contains the files/instructions on reproducing an issue that we've seen on EKS IPv6 clusters using AWS VPC CNI.

Kubernetes Versions:
- 1.28 (eks.7) w/ kube-proxy v1.28.2-eksbuild.2
- 1.29 (eks.1) w/ kube-proxy v1.29.0-eksbuild.2

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

It may take a few attempts to reproduce errors, but the observed behavior is that large simultaneous downloads stall out and eventually we receive a "connection reset by peer" error. Sometimes, we also see TLS connection errors and DNS resolution errors, which cause the download to immediately error out.

Connection reset/timeout errors tend to be more prominent during business hours (when there's likely more network traffic across the VPC / within the cluster).

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

Sometimes we see errors around establishing connections over HTTPS:
```
test9 | Connecting to embed-ssl.wistia.com (embed-ssl.wistia.com)|2600:9000:244d:7800:1e:c86:4140:93a1|:443... connected.
test9 | Unable to establish SSL connection.
test9 | exit status 4
```

```
test3 | Resolving embed-ssl.wistia.com (embed-ssl.wistia.com)... failed: Try again.
test3 | wget: unable to resolve host address 'embed-ssl.wistia.com'
test3 | exit status 4
```
