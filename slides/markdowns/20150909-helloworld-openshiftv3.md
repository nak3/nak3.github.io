background-image: url("./image/rhte-logo_grey_font_text.png")
class: center, middle

# Hacking OpenShift v3


## Kenjiro Nakayama

---
# Outline

- Hello World, OpenShift v3

- Behind the HelloWorld

  - Resources

  - S2I Build process

  - Containers in the background

---
# Who am I?

- Kenjiro Nakayama

- From Tokyo, Japan

- OpenShift Support Engineer

- Fedora Project Package maintainer

- Emacs committer

---
class: center, middle

# Hello World in 3min

---
# Hello World in 3min

--
- **From**

  - OpenShift enterprise/origin source code build

--
- **Via**

  - Router & Registry setup

--
- **To**

  - S2I build from github application

--
- **By**

  - One script


---
class: center, middle

# Demo

---
# Hello World in 3min

--
- Step 1. Download source code and build script

```area
cd /tmp
git clone https://github.com/openshift/origin.git -b master
wget https://raw.githubusercontent.com/nak3/openshift-local-setup/master/openshift-local-setup.sh
```

--

- Step 2. Run script (Origin Build & Run & Setup)

```area
bash openshift-local-setup.sh
```

---
# Hello World in 3min

- Setp 3. Setup client

```area
export PATH="/tmp/origin/_output/local/go/bin/:$PATH"
alias oc="oc --config=/tmp/origin/openshift.local.config/master/admin.kubeconfig"
alias oadm="oadm --config=/tmp/origin/openshift.local.config/master/admin.kubeconfig"
source /tmp/origin/rel-eng/completions/bash/oc
```
--

- Step 4. Create project

```area
oadm policy add-role-to-user admin joe
oc login -u joe -p foo
oc new-project demoproject
```
---
# Hello World in 3min

- Step 5. Create new application

```area
oc new-app https://github.com/nak3/helloworld-v3.git
```

--
- Step 6. Hello World!

```area
curl 172.30.36.155:8080
Hello World!
```
---
# Hello World in 3min

- Step -7. Routing test

```areaa
oc expose service sti-php --hostname=helloworld-v3.example.com
sudo echo "192.168.122.3  helloworld-v3.example.com" >> /etc/hosts
curl helloworld-v3.example.com
Hello World!
```
---
# Hello World in 3min

- You can download the setup script

  - https://github.com/nak3/openshift-local-setup

  - Needs Go > 1.4 and docker > 1.6

  - Tested with RHEL7/Fedora21/CentOS7/Ubuntu14.04

---
class: center, middle

# What happend behind?

---
class: center, middle

## Today's focus

```bash
$ oc new-app https://github.com/nak3/helloworld-v3.git
```
---
class: center, middle

# Quiz

---
# Q1. How many containers did I make?

### A.  1

### B.  3

### C.  6

---
# Q2. Correct order

a. Running S2I build

b. Created Replication Controller

c. Created New Build

d. Created Image Stream

---
# Behind the Hello World

![helloworld-process](./image/helloworld-process-updated.png)

---
class: center, middle

# 1. Resources

---
# 1. Resources

![helloworld-process](./image/helloworld-process-updated-resource.png)

---
# What resources are created

- ImageStream

- BuildConfig

- DeploymentConfig

- Service

- Build

- Replication Controller

- Pod

---
# Useful information for Resources

- `oc get event`

- `oc --loglevel=8`

- `journalctl -u {openshift-master,openshift-node,docker} --no-pager -l -a`

- (Advanced) web console debug ([KCS#1504213](https://access.redhat.com/solutions/1504213) / [origin doc](https://github.com/openshift/origin/blob/master/assets/README.md#enable--disable-console-log-output))
```area
localStorage["OpenShiftLogLevel.main"] = "<log level>";
```

- (Advanced) `debug.PrintStack()` in source code

  - GDB was not good at this moment...

---
# Tips for resources debug/hack

- ###Divide resources###

- Example steps...

  - step-1. Output to json
```area
oc new-app https://github.com/nak3/helloworld-v3.git -o json > resources.json
```

  - step-2. Make only one resource in json file
```area
cat resources.json | jq .items[0] > imagestream.json
```

  - step-3. Test with one resource
```area
oc create -f imagestream.json
```

  - step-4. Check the log

---
class: center, middle

# Demo

---
# Demo

![helloworld-process](./image/helloworld-process-updated-resource.png)

---
class: center, middle

# 2. S2I Build process

---
# 2. S2I Build process

![helloworld-process](./image/helloworld-process-updated-build.png)

---
# Tips for S2I building debug/hack

- See build-logs with [BUILD_LOGLEVEL 5](https://docs.openshift.com/enterprise/3.0/dev_guide/builds.html#canceling-a-build) in BuildConfig

```area
  "sourceStrategy": {
    ...
    "env": [
      {
        "Name": "BUILD_LOGLEVEL",
        "Value": "5"
      }
    ...
```

--

- Test by custom builder

  - Docker push failed

  - git clone failed

---
# Custom-builder-debug

- You can download the sample from here

  - https://github.com/nak3/custom-builder-for-test/tree/master/custom-builder-debug

- Or just run for test

```area
oc create -f \
   https://raw.githubusercontent.com/nak3/custom-builder-for-test/master
                           /custom-builder-debug/assets/ruby-hello-world.json
```
---
# Custom-builder-debug

- Output all information in the builder image

```area
$ oc build-logs ruby-hello-world-1
DEBUG:
DEBUG: Output environment value
DEBUG: ------------------------
SOURCE_URI=https://github.com/openshift/ruby-hello-world.git
OPENSHIFT_CUSTOM_BUILD_BASE_IMAGE=nak3/custom-builder-debug@sha256:a4c85322eaf87040e71fe7b8b33dec504252924562fc3d09ecde492ae73284f6
HOSTNAME=ruby-hello-world-1-build
RUBY_HELLO_WORLD_SERVICE_PORT_8080_TCP=8080
KUBERNETES_PORT_443_TCP_PORT=443
KUBERNETES_PORT=tcp://172.30.0.1:443

  ...

```
---
class: center middle

# Demo

---
class: center middle

# Just one more thing

---
class: center middle

#3. Containers in the background

---
#3. Containers in the background

![helloworld-process](./image/helloworld-process-updated-container.png)

---

# Containers in the background

- ose-pod * 3

- ose-sti-builder / ose-docker-builder / ose-custom-builder

- ose-deployer

- Pod (Hello World)


```area
[root@ose3-node1 ~]# docker ps -a
2a2959d29670        172.30.255.87:5000/demo/helloworld-v3@sha256:e003bb1848e9e6519be1bb200d3e48358f4f56dbebb7f521f5284553a29d9afa   "/usr/local/sti/run"   3 minutes ago       Up 3 minutes                            k8s_helloworld-v3.e81e7ee8_helloworld-v3-1-w4z1b_demo_1810bbc1-522f-11e5-aab3-525400b33d1d_cf369d77
0b86e1d93a0a        openshift3/ose-pod:v3.0.1.0                                                                                     "/pod"                 About a minute ago   Up About a minute                                   k8s_POD.3190e4a2_helloworld-v3-1-w4z1b_demo_1810bbc1-522f-11e5-aab3-525400b33d1d_e96a5c40
a3f7003473bb        openshift3/ose-deployer:v3.0.1.0                                                                                "/usr/bin/openshift-   About a minute ago   Exited (0) 59 seconds ago                           k8s_deployment.3d017e40_helloworld-v3-1-deploy_demo_14f89e40-522f-11e5-aab3-525400b33d1d_5aaeddd7
fe2e3a355f65        openshift3/ose-pod:v3.0.1.0                                                                                     "/pod"                 About a minute ago   Exited (0) 49 seconds ago                           k8s_POD.892ec37e_helloworld-v3-1-deploy_demo_14f89e40-522f-11e5-aab3-525400b33d1d_b272ae73
5debbb36cab1        openshift3/ose-sti-builder:v3.0.1.0                                                                             "/usr/bin/openshift-   3 minutes ago        Exited (0) About a minute ago                       k8s_sti-build.2e52d9e_helloworld-v3-1-build_demo_add8c71e-522e-11e5-aab3-525400b33d1d_6e97fd75
bf2935e0bd41        openshift3/ose-pod:v3.0.1.0                                                                                     "/pod"                 4 minutes ago        Exited (0) About a minute ago
```
---
# What is ose-pod?

- Same with kubernetes's "pause" container

- Each pod has one `ose-pod`

- Only two functions

  - IPC control - [images/pod/pod.go](https://github.com/openshift/origin/blob/master/images/pod/pod.go)

  - Holding network endpoint

```bash
# docker ps |grep helloworld
cec670f80d07        172.30.196.37:5000/default/helloworld-v3@sha256:df615cfb8c1a42703ff1d43aa39c582dab9be78c28b3509f4b6d06a1d4a0d84d   "/usr/local/sti/run"   About an hour ago   Up About an hour                        k8s_helloworld-v3.9f268c47_helloworld-v3-1-45xn7_default_f5ea369e-506b-11e5-b1ae-525400b5bdf9_d940ad00
35a2d2271a00        openshift/ose-pod:v1.0.5                                                                                        "/pod"                 About an hour ago   Up About an hour                        k8s_POD.8786f5dc_helloworld-v3-1-45xn7_default_f5ea369e-506b-11e5-b1ae-525400b5bdf9_901ce512
```

```sh
# docker inspect cec670f80d07 |grep NetworkMode
        "NetworkMode": "container:35a2d2271a00bf44ee4dc684a785a02157ba587df54a390ca226ee36fef5230f",
# docker inspect 35a2d2271a00 |grep NetworkMode
        "NetworkMode": "bridge",
```

---
# builder

- The builder container was chosen by build strategy

   - source strategy ... sti-builder

   - docker strategy ... docker-builder

   - custom strategy ... custom-builder

- build process is run in the builder container

- ≒ `helloworld-v3-1-build (pod)`

---
# deployer

- Handling deployment strategy

 - eg. RecreateDeploymentStrategy, RollingDeploymentStrategy...

 - (Ref: OpenShift documentation - [Deployments](https://docs.openshift.com/enterprise/3.0/dev_guide/deployments.html))

- Short life! Check the container logs

```area
# docker ps -a |grep helloworld-v3-1-deploy
f8a529829f6c        openshift/ose-deployer:v1.0.5                                                                                   "/usr/bin/openshift-   41 minutes ago      Exited (0) 41 minutes ago                          k8s_deployment.d2c4869c_helloworld-v3-1-deploy_default_4ce7e547-50a8-11e5-89a3-525400b5bdf9_ba74c4e8
# docker logs f8a529829f6c
I0901 12:52:30.470260       1 deployer.go:195] Deploying default/helloworld-v3-1 for the first time (replicas: 1)
I0901 12:52:30.471636       1 recreate.go:112] Scaling default/helloworld-v3-1 to 1 before performing acceptance check
I0901 12:52:32.545151       1 recreate.go:117] Performing acceptance check of default/helloworld-v3-1
I0901 12:52:32.545719       1 lifecycle.go:279] Waiting 600 seconds for pods owned by deployment "default/helloworld-v3-1" to become ready (checking every 1 seconds; 0 pods previously accepted)
I0901 12:52:42.546422       1 lifecycle.go:300] All pods ready for default/helloworld-v3-1
I0901 12:52:42.546507       1 recreate.go:143] Deployment helloworld-v3-1 successfully made active
```

---
# Summary

- Hello World in 3min

 - Build and deploy test quickly

- Behind the Hello World

  - Resources

  - S2I build process

  - Containers in the background

---
# Reference

- OpenShift v3 Documentation

  - https://docs.openshift.com/enterprise/3.0/getting_started/administrators.html

- OpenShift Origin Source Code

  - https://github.com/openshift/origin

- source-to-image Source Code

  - https://github.com/openshift/source-to-image

- ose-pod / origin-pod Source Code

  - https://github.com/openshift/origin/blob/master/images/pod/pod.go

---
# Reference

- openshift-local-setup

  - https://github.com/nak3/openshift-local-setup

- custom-builder-debug

  - https://github.com/nak3/custom-builder-for-test/tree/master/custom-builder-debug

---
class: center middle
# End
