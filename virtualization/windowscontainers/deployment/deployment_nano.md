---
title: Deploy Windows Containers on Nano Server
description: Deploy Windows Containers on Nano Server
keywords: docker, containers
author: neilpeterson1
manager: timlt
ms.date: 08/23/2016
ms.topic: article
ms.prod: windows-containers
ms.service: windows-containers
ms.assetid: b82acdf9-042d-4b5c-8b67-1a8013fa1435
---

# Container host deployment - Nano Server

**This is preliminary content and subject to change.** 

This document will step through a very basic Nano Server deployment with the Windows container feature. This is an advanced topic and assumes a general understanding of Windows and Windows containers. For an introduction to Windows containers, see [Windows Containers Quick Start](../quick_start/quick_start.md).

## Prepare Nano Server

The following section will detail the deployment of a very basic Nano Server configuration. For a more through explanation of deployment and configuration options for Nano Server, see [Getting Started with Nano Server] (https://technet.microsoft.com/en-us/library/mt126167.aspx).

### Create Nano Server VM

First download the Nano Server evaluation VHD from [this location](https://msdn.microsoft.com/en-us/virtualization/windowscontainers/nano_eula). Create a virtual machine from this VHD, start the virtual machine, and connect to it using the Hyper-V connect option, or equivalent based on the virtualization platform being used.

### Create Remote PowerShell Session

Because Nano Server does not have interactive log on capabilities, all management will be completed from a remote system using PowerShell.

Add the Nano Server system to trusted hosts of the remote system. Replace the IP Address with the IP Address of the Nano Server.

```none
Set-Item WSMan:\localhost\Client\TrustedHosts 192.168.1.50 -Force
```

Create the remote PowerShell session.

```none
Enter-PSSession -ComputerName 192.168.1.50 -Credential ~\Administrator
```

When these steps have been completed, you will be in remote PowerShell session with the Nano Server system. The remainder of this document, unless noted otherwise, will take place from the remote session.


## Install Container Feature

The Nano Server package management provider allows roles and features to be installed on Nano Server. Install the provider using this command.

```none
Install-PackageProvider NanoServerPackage
```

After the package provide has been installed, install the container feature.

```none
Install-NanoServerPackage -Name Microsoft-NanoServer-Containers-Package
```

The Nano Server host will need to be re-booted after the container features has been installed. 

```none
Restart-Computer
```

Once it is back up, re-establish the remote PowerShell connection.

## Install Docker

The Docker Engine is required in order to work with Windows containers. Install the Docker Engine using these steps.

First, ensure that the Nano Server firewall has been configured for SMB. This can be completed by running this command on the Nano Server host.

```none
Set-NetFirewallRule -Name FPS-SMB-In-TCP -Enabled True
```

Create a folder on the Nano Server host for the Docker executables.

```none
New-Item -Type Directory -Path $env:ProgramFiles'\docker\'
```

Download the Docker Engine and client and copy these into 'C:\Program Files\docker\' of the container host. 

> Nano Server does not currently support `Invoke-WebRequest`. the download will need to be completed on a remote system, and the files copied to the Nano Server host.

```none
Invoke-WebRequest "https://get.docker.com/builds/Windows/x86_64/docker-1.12.0.zip" -OutFile .\docker-1.12.0.zip -UseBasicParsing
```

Extract the downloaded package. Once completed you will have a directory containing both **dockerd.exe** and **docker.exe**. Copy both of these to the **C:\Program Files\docker\** folder in the Nano Server container host. 

```none
Expand-Archive .\docker-1.12.0.zip
```

Add the Docker directory to the system path on the Nano Server.

> Make sure to switch back to the remote Nano Server session.

```none
# For quick use, does not require shell to be restarted.
$env:path += “;C:\program files\docker”

# For persistent use, will apply even after a reboot.
setx PATH $env:path /M
```

Install Docker as a Windows service.

```none
dockerd --register-service
```

Start the Docker service.

```none
Start-Service Docker
```

## Install Base Container Images

Base OS images are used as the base to any Windows Server or Hyper-V container. Base OS images are available with both Windows Server Core and Nano Server as the underlying operating system and can be installed using `docker pull`. For detailed information on Windows container images, see [Managing Container Images](../management/manage_images.md).

To download and install the Nano Server base image, run the following:

```none
docker pull microsoft/nanoserver
```

> At this time, only the Nano Server base image is compatible with a Nano Server container host.

## Manage Docker on Nano Server

For the best experience, and as a best practice, manage Docker on Nano Server from a remote system. In order to do so, the following items need to be completed.

### Prepare Container Host

Create a firewall rule on the container host for the Docker connection. This will be port `2375` for an unsecure connection, or port `2376` for a secure connection.

```none
netsh advfirewall firewall add rule name="Docker daemon " dir=in action=allow protocol=TCP localport=2375
```

Configure the Docker Engine to accept incoming connection over TCP.

First create a `daemon.json` file at `c:\ProgramData\docker\config\daemon.json` on the Nano Server host.

```none
new-item -Type File c:\ProgramData\docker\config\daemon.json
```

Next, run the following command to add connection configuration to the `daemon.json` file. This configures the Docker Engine to accept incoming connections over TCP port 2375. This is an unsecure connection and is not advised, but can be used for isolated testing. For more information on securing this connection, see [Protect the Docker Daemon on Docker.com](https://docs.docker.com/engine/security/https/).

```none
Add-Content 'c:\programdata\docker\config\daemon.json' '{ "hosts": ["tcp://0.0.0.0:2375", "npipe://"] }'
```

Restart the Docker service.

```none
Restart-Service docker
```

### Prepare Remote Client

On the remote system where you will be working, download the Docker client.

```none
Invoke-WebRequest "https://get.docker.com/builds/Windows/x86_64/docker-1.12.0.zip" -OutFile "$env:TEMP\docker-1.12.0.zip" -UseBasicParsing
```

Extract the compressed package.

```none
Expand-Archive -Path "$env:TEMP\docker-1.12.0.zip" -DestinationPath $env:ProgramFiles
```

Run the following two commands to add the Docker directory to the system path.

```none
# For quick use, does not require shell to be restarted.
$env:path += ";c:\program files\docker"

# For persistent use, will apply even after a reboot. 
[Environment]::SetEnvironmentVariable("Path", $env:Path + ";C:\Program Files\Docker", [EnvironmentVariableTarget]::Machine)
```

Once completed the remote Docker host can be accessed with the `docker -H` parameter.

```none
docker -H tcp://<IPADDRESS>:2375 run -it nanoserver cmd
```

An environmental variable `DOCKER_HOST` can be created which will remove the `-H` parameter requirement. The following PowerShell command can be used for this.

```none
$env:DOCKER_HOST = "tcp://<ipaddress of server>:2375"
```

With this variable set, the command would now look like this.

```none
docker run -it nanoserver cmd
```

## Hyper-V Container Host

In order to deploy Hyper-V containers, the Hyper-V role will be required on the container host. For more information on Hyper-V containers, see [Hyper-V Containers](../management/hyperv_container.md).

If the Windows container host is itself a Hyper-V virtual machine, nested virtualization will need to be enabled. For more information on nested virtualization, see [Nested Virtualization](https://msdn.microsoft.com/en-us/virtualization/hyperv_on_windows/user_guide/nesting).


Install the Hyper-V role on the Nano Server container host.

```none
Install-NanoServerPackage Microsoft-NanoServer-Compute-Package
```

The Nano Server host will need to be re-booted after the Hyper-V role has been installed.

```none
Restart-Computer
```
