# UCP with Server 1709 testing

UCP on Azure

Create a Ubuntu 16.04 VM w static private IP address
Create new network security group with required inbound ports

443
2376-2377
4789
7946
12376,12379
12380-12387

Install Docker
https://docs.docker.com/engine/installation/linux/docker-ce/ubuntu/

```
sudo -i
apt-get update
apt-get install apt-transport-https ca-certificates curl software-properties-common
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | apt-key add -
add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
apt-get update
apt-cache show docker-ce
apt-get install docker-ce=17.06.*
docker version
docker run hello-world
```

Install UCP

https://docs.docker.com/datacenter/ucp/2.2/guides/admin/install/ 
Host address = resource group subnet (ie 172.16.8.6)
Licenses pinned to #product-ee channel
Add public IP to SAN list

Create "WS1709 with Containers" VM from Azure MarketPlace

Use same network security group as above

Install EE Preview
```
stop-service docker
uninstall-package Docker -PackageProvider DockerMsftProvider

Install-Module DockerProvider
Install-Package Docker -Providername DockerProvider -RequiredVersion preview
```
For use with Windows Server 2016 (RS1) images set daemon to use hyper-v isolation
https://docs.microsoft.com/en-us/virtualization/windowscontainers/deploy-containers/version-compatibility
```
Create c:\ProgramData\docker\config\daemon.json

{
    "exec-opts":["isolation=hyperv"],
    "debug": true    
}
```
```
dockerd -D
docker version
```

Install UCP agent and run script

```
docker image pull docker/ucp-agent-win:2.2.4
docker image pull docker/ucp-dsinfo-win:2.2.4
docker container run --rm docker/ucp-agent-win:2.2.4 windows-script | powershell -noprofile -noninteractive -command 'Invoke-Expression -Command $input'
```

Add Windows node in UCP and join swarm via provided URL

### Known issues

1 - Warning in UCP windows-script output
```
Testing for required windows updates  = [System.Version]::Parse 10.0.16299.15  = [System.Version]::Parse 10.0.14393.1066 if False       Write-Host "System is missing a required update. Please check windows updates or apply this KB4015217: http://www.catalog.update.microsoft.com/Search.aspx?q=KB4015217 before adding this node to your UCP cluster" -ForegroundColor yellow  Write-Host Setting up Docker daemon to listen on port 2376 with TLS

Generating new certs at C:\ProgramData\docker\daemoncerts
Restarting Docker daemon
Successfully set up Docker daemon
Opening port 2376 in the Windows firewall for inbound traffic
Opening port 12376 in the Windows firewall for inbound traffic
Opening port 2377 in the Windows firewall for inbound traffic
Opening port 4789 in the Windows firewall for inbound and outbound traffic
Opening port 7946 in the Windows firewall for inbound and outbound traffic
```

2 - UCP agent logs feature not working for 1709 workers

----------

# VIP testing with Server 1709
## Mixed swarm
Create a 3 node swarm 
* 1 x Ubuntu 16.04 master running Docker EE 17.06
* 2 x Windows Server 1709 workers running EE Preview-3

Deploy two services, each will get a VIP address
```
docker network create overlay1 --driver overlay
docker service create --name s1 --replicas 2 --network overlay1 --constraint node.platform.os==windows microsoft/iis
docker service create --name s2 --replicas 2 --network overlay1 --constraint node.platform.os==windows microsoft/iis
```

Find a container ID for each task (run on workers)
```
docker ps --format "{{.ID}}: {{.Names}}"
```

Verify connectiviy between services s1 and s2 via VIP on overlay network. On worker running a task for service s1:
```
docker exec -it <ID of s1 container> powershell
Invoke-WebRequest -Uri http://s2 -UseBasicParsing
```
When the Linux master is running 17.06 the following failure occurs. When running 17.10 the operation succeeds as expected.
```
Invoke-WebRequest : Unable to connect to the remote server
At line:1 char:1
+ Invoke-WebRequest -Uri http://s2 -UseBasicParsing
+ ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
    + CategoryInfo          : InvalidOperation: (System.Net.HttpWebRequest:HttpWebRequest) [Invoke-WebRequest], WebExc
   eption
    + FullyQualifiedErrorId : WebCmdletWebResponseException,Microsoft.PowerShell.Commands.InvokeWebRequestCommand
```
The same test but using the VIP IP address directly also fails.

## Native Windows swarm
Create a 2 node Windows Server 1709 swarm running EE Preview-3
Use above image
Use above commands to create overlay network, and deploy services

Verify connectivity between services s1 and s2 via VIP on overlay network. On worker running a task for service s1:
```
docker exec -it <ID of s1 container> powershell
Invoke-WebRequest -Uri http://s2 -UseBasicParsing
```
Returns a ```200``` from the default IIS website:
```
StatusCode        : 200
StatusDescription : OK
Content           : <!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Strict//EN"
                    "http://www.w3.org/TR/xhtml1/DTD/xhtml1-strict.dtd">
                    <html xmlns="http://www.w3.org/1999/xhtml">
                    <head>
                    <meta http-equiv="Content-Type" cont...
RawContent        : HTTP/1.1 200 OK
                    Accept-Ranges: bytes
                    Content-Length: 703
                    Content-Type: text/html
                    Date: Wed, 08 Nov 2017 02:01:14 GMT
                    ETag: "f2f79a40d951d31:0"
                    Last-Modified: Mon, 30 Oct 2017 23:46:05 GMT
                    Serve...
Forms             :
Headers           : {[Accept-Ranges, bytes], [Content-Length, 703], [Content-Type, text/html], [Date, Wed, 08 Nov 2017
                    02:01:14 GMT]...}
Images            : {@{outerHTML=<img src="iisstart.png" alt="IIS" width="960" height="600" />; tagName=IMG;
                    src=iisstart.png; alt=IIS; width=960; height=600}}
InputFields       : {}
Links             : {@{outerHTML=<a href="http://go.microsoft.com/fwlink/?linkid=66138&amp;clcid=0x409"><img
                    src="iisstart.png" alt="IIS" width="960" height="600" /></a>; tagName=A;
                    href=http://go.microsoft.com/fwlink/?linkid=66138&amp;clcid=0x409}}
ParsedHtml        :
RawContentLength  : 703
```

#Ingress publishing testing with Server 1709
## Mixed swarm

```docker service create --name s3 --replicas 2 --network overlay1 -p 8080:80 --constraint node.platform.os==windows microsoft/iis
```
Ensure port 8080 is open in Azure network security group used by the VMs in the swarm
Browse to ```http://<Public IP address of any VM in the swarm>:8080```
Default IIS website should be displayed

------------
Find the VIP addresses for a service
```
docker service inspect s1
```
------------
Opening ports in Windows firewall
```
netsh firewall add portopening TCP 2377 "Port 2377"
netsh firewall add portopening TCP 2376 "Port 2376"
netsh firewall add portopening TCP 7496 "Port 7496"
```
------------
Tailing docker daemon logs on Ubuntu
```
journalctl -f -u docker.service
```
------------
iptables configuration
```
#!/bin/bash

ports='443 12386 12387 12379 12385 12376 12384 12381 12383 12380 2376 2377 12382 12386 4789 7946'

for port in $ports
do
   iptables -A INPUT -p udp --dport $port -j ACCEPT
   iptables -A INPUT -p tcp --dport $port -j ACCEPT
done
iptables-save > /etc/iptables.conf
add iptables-restore /etc/iptables.conf to /etc/rc.local
```
------------
Create a wrapper image to run IIS
```
FROM microsoft\iis
EXPOSE 80
```

Build and push
```
docker build .
docker tag <image ID> carlfischer/cfiis
docker login --username carlfischer
docker push carlfischer/cfiis
```
See https://github.com/docker/saas-mega/issues/3389. To workaround, build the image on both Windows workers.



