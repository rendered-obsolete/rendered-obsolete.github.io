## Building with Docker

[Build with docker](https://github.com/zplus/thrift/blob/master/build/docker/README.md).



```
DOCKER_REPO=thrift/thrift-build DISTRO=ubuntu-bionic build/docker/refresh.sh
```
```
docker run -v $(pwd):/thrift/src -it ubuntu-bionic /bin/bash
```
Fails with:
```
Unable to find image 'ubuntu-bionic:latest' locally
docker: Error response from daemon: pull access denied for ubuntu-bionic, repository does not exist or may require 'docker login'.
See 'docker run --help'.
```
Scroll way up to the top of `refresh.sh` and you'll find: 
```
~/projects/thrift/build/docker/ubuntu-bionic ~/projects/thrift
ubuntu-bionic: Pulling from thrift/thrift-build
Digest: sha256:05e8bc817825fd2b262263be64184b87423278dc00037e7fadfed336ce75e74c
Status: Downloaded newer image for thrift/thrift-build:ubuntu-bionic
build/docker/refresh.sh: line 40: sha512sum: command not found
```

https://stackoverflow.com/questions/7500691/rvm-sha256sum-nor-shasum-found