# ConductR Sandbox

[![Build Status](https://api.travis-ci.org/typesafehub/sbt-conductr-sandbox.png?branch=master)](https://travis-ci.org/typesafehub/sbt-conductr-sandbox)

## Introduction

sbt-conductr-sandbox aims to support the running of a Docker-based ConductR cluster in the context of a build. The cluster can then be used to assist you in order to verify that endpoint declarations and other aspects of a bundle configuration are correct. The general idea is that this plugin will also support you when building your project on CI so that you may automatically verify that it runs on ConductR.

## Prerequisites
- [Docker](https://www.docker.com/)

## Usage

The version of the ConductR Developer Sandbox is available gratis during development with registration at Typesafe.com. Please visit the [ConductR Developer](http://www.typesafe.com/product/conductr/developer) page on Typesafe.com for the current image version and licensing information. Use of ConductR for production purposes requires the purchase of Typesafe ConductR. You <strong>must</strong> specify the current `imageVersion which you can obtain from the [ConductR Developer](http://www.typesafe.com/product/conductr/developer) page.

### Docker

To get started quickly, sbt-conductr-sandbox is using a pre-packaged Docker image which has ConductR already installed. In order to use sbt-conductr-sandbox please install [Docker](https://www.docker.com/) and the `docker-machine` CLI.

Verify the installation by entering the following command into the terminal:

```bash
docker-machine
Usage: docker-machine [OPTIONS] COMMAND [arg...]
...
```

Afterwards, start the Docker VM with:

```bash
docker-machine start default
```

This plugin uses the the VirtualBox VM `default` which is the one Docker uses by default.

### Configuring ConductR sandbox

Declare the sbt-conductr-sandbox plugin in the `plugins.sbt` of your project:

```scala
addSbtPlugin("com.typesafe.conductr" % "sbt-conductr-sandbox" % "1.1.2")
```

Nothing more is required to enable the plugin.

### Starting ConductR sandbox

To run the sandbox environment run the following command inside the sbt session:

```scala
sandbox run
```

> Note that the ConductR cluster will take a few seconds to become available and so any initial command that you send to it may not work immediately.

Given the above you will then have a ConductR process running in the background (there will be an initial download cost for Docker to download the image from the public Docker registry).

This plugin automatically enables `sbt-conductr` for your project so you can use the `conduct info` and other `conduct` commands. These commands will automatically communicate with the Docker cluster managed by the sandbox.

#### Starting multiple ConductR nodes

By default `sandbox run` is starting a ConductR cluster with one node. Specify the option `--nr-of-containers` to during startup to set the number of nodes in the ConductR cluster:
 
```scala
sandbox run --nr-of-containers 3
```

#### Adding / stopping ConductR nodes

To add or stop nodes in the ConductR cluster use the `sandbox run`. First, let's start a new ConductR cluster with two nodes:

```
sandbox run --nr-of-containers 2
```

Now, let's start one additional node:
 
```
sandbox run --nr-of-containers 3
```

The first two nodes have been untouched and one additional node has been started. Finally, let's stop two nodes:

```
sandbox run --nr-of-containers 1
```

The last two nodes have been stopped in our cluster, i.e. `cond-2` and `cond-1` have been stopped while `cond-0` is still running.

#### Starting with ConductR features

To enable optional ConductR features for your sandbox specify the `--with-features` option during startup, e.g.:
    
```scala
sandbox run --with-features visualization logging
[info] Running ConductR...
[info] Running container cond-0 exposing 192.168.59.103:9000 192.168.59.103:9909...
```

Check out the [Features](#features) section for more information.

### Debugging application in ConductR sandbox

It is possible to debug your application inside of the ConductR sandbox:

1. Start ConductR sandbox in debug mode:

    ```scala
    sandbox debug
    ```
2. This exposes a debug port to Docker and the ConductR container. By default the debug port is set to `5005` which is the default port for remote debugging in IDEs such as IntelliJ. If you are happy with the default setting skip this step. Otherwise specify another port in the `build.sbt`:

    ```scala
    SandboxKeys.debugPort := 5432
    ```    
4. Load and run your bundle to the sandbox:

    ```scala
    conduct load <HIT THE TAB KEY AND THEN RETURN>
    conduct run my-app
    ```

You should be now able to remotely debug the bundle inside the ConductRs sandbox with your favorite IDE.    

### Stopping ConductR sandbox

To stop the sandbox use: 

```scala
sandbox stop
```

## Features

> In order to use the following features you should ensure that the machine that runs Docker has enough memory, typically at least 2GB. VM configurations such as those provided via `docker-machine` and Oracle's VirtualBox can be configured like so:
> ```
> docker-machine stop default
> VBoxManage modifyvm default --memory 2048
> docker-machine start default
> ```

ConductR provides additional features which can be optionally enabled:

Name          | CondutR ports | Docker ports | Description
--------------|---------------|-------------|------------
visualization | 9999          | 9909        | Provides a web interface to visualize the ConductR cluster together with deployed and running bundles.  After enabling the feature, access it at http://{docker-host-ip}:9909.
logging       | 9200, 5601    | 9200, 5601  | Consolidates the logging output of ConductR itself and the bundles that it executes. To view the consolidated log messsages run `conduct logs {bundle-name} or access the Kibana UI at http://{docker-host-ip}:5601.

## Docker Container Naming

Each node of the Docker cluster managed by the sandbox is given a name of the form `cond-n` where `n` is the node number starting at zero. Thus `cond-0` is the first node (and the only node given default settings).

## Port Mapping Convention

The following ports are exposed to the ConductR and Docker containers:
- `BundleKeys.endpoints` of `sbt-bundle`
- `SandboxKeys.ports`
- `SandboxKeys.debugPort` if sandbox is started in debug mode

### Example
Your application defines these settings in the `build.sbt`:

```
lazy val root = (project in file(".")).enablePlugins(JavaAppPackaging)

BundleKeys.endpoints := Map("sample-app" -> Endpoint("http", services = Set(uri("http://:9000"))))
SandboxKeys.imageVersion in Global := "1.0.12"
SandboxKeys.ports in Global := Set(1111)
SandboxKeys.debugPort := 5095
``` 

We specify that the web application should serve traffic on port 9000. Additionally we expose port 1111. The debug port gets only mapped if we start the ConductR sandbox cluster in debug mode with `sandbox debug`.

Now we create a ConductR cluster with 3 nodes:

```
sandbox run --nr-of-containers 3
```

These settings result in the following port mapping:

Docker container | ConductR port | Docker internal port | Docker public port
-----------------|---------------|----------------------|-------------------
cond-0           | 9000          | 9000                 | 9000
cond-1           | 9000          | 9000                 | 9010
cond-2           | 9000          | 9000                 | 9020
cond-0           | 1111          | 1111                 | 1111
cond-1           | 1111          | 1111                 | 1121
cond-2           | 1111          | 1111                 | 1131
cond-0           | 5095          | 5095                 | 5095
cond-1           | 5095          | 5095                 | 5005
cond-2           | 5095          | 5095                 | 5015

Each specified port is mapped to a unique public Docker port in order to allow multiple nodes within the sandbox cluster. By convention, the first node is using the ConductR port as the public Docker port, e.g. 1111 becomes 1111. The following nodes using a different public Docker port by increasing the second last digit of the ConductR port by the node number, e.g. the second node is mapping the port 1111 to 1121, the third is mapping the port 1111 to 1131. 
The web application becomes available on the Docker host IP address. The sandbox cluster is configured with a proxy and will automatically route requests to the correct instances that you have running in the cluster. Therefore any one of the addresses with the 9000, 9010 or 9020 ports will reach your application.

As a convenience, `sandbox run` and `sandbox debug` is reporting each of the above mappings along with the IP addresses:

```
[info] Starting ConductR nodes..
[info] Starting container cond-0 exposing 192.168.99.100:9000, 192.168.99.100:1111, 192.168.99.100:5095...
[info] f7f030f8b97cc375642f5a0321654594547b1e7d3a327c6f120715ba3199dda0
[info] Starting container cond-1 exposing 192.168.99.100:9010, 192.168.99.100:1121, 192.168.99.100:5005...
[info] 7f2caf1053810465589ac9710e0676a39d6828687c06020fa2d4a80ba815d80a
[info] Starting container cond-2 exposing 192.168.99.100:9020, 192.168.99.100:1131, 192.168.99.100:5015...
[info] 5644465b394617fb8d18bdb64ce488257776eea4437d7d1d205b481b775ab42a
```

## Roles

ConductR uses roles to load and scale bundles to the respective nodes. For more information about it please refer to [Managing services documentation](http://conductr.typesafe.com/docs/1.0.x/ManageServices#Roles).

`sbt-conductr-sandbox` automatically collects the bundle roles specified with `BundleKeys.roles` and adds them to each ConductR node. Therefor, by default your bundles can be loaded and scaled to all ConductR nodes. 
It is also possible to provide a custom role configuration by specifying the `SandboxKeys.conductrRoles` setting: 

```scala
SandboxKeys.conductrRoles in Global := Seq(Set("muchMem", "myDB"))
```

This is only adding the roles `muchMem` and `myDB` to all ConductR nodes. The bundle roles are ignored in case `SandboxKeys.conductrRoles` is specified. Assigning different roles to nodes is also possible by specifying a set for each of the nodes:

```scala
SandboxKeys.conductrRoles in Global := Seq(Set("muchMem"), Set("myDB"))
```

In the above example the bundles with the role `muchMem` would only be loaded and scaled to the first node and bundles with the role `myDB` only to the second node.

If the `--nr-of-containers` startup option is larger than the size of the `conductrRoles` sequence then the specified roles are subsequently applied to the remaining nodes.

```scala
SandboxKeys.conductrRoles in Global := Seq(Set("muchMem", "frontend"), Set("myDB"))
```

```scala
sandbox run --nr-of-containers 4
```

This would load and scale all bundles with the roles `muchMem` and `frontend` to the first and third node and bundles with the role `myDB` to the second and fourth node.

## Functions

There is a `ConductRSandbox.runConductRs` function for starting the sandbox environment programmatically. This is useful for setting up test environments. Conversely there is a `ConductRSandbox.stopConductRs` function call can be made to close down the sandbox environment programmatically. The following listing shows how these functions can be used within an sbt integration test:

```scala
testOptions in IntegrationTest ++= Seq(
  Tests.Setup { () =>
    ConductRSandbox.stopConductRs(streams.value.log)
    ConductRSandbox.runConductRs(
      1,
      Seq.empty,
      Set.empty,
      Map.empty,
      (conductrImage in Global).value,
      (conductrImageVersion in Global).value,
      Set.empty,
      streams.value.log,
      "info")
  },

  Tests.Cleanup (() => ConductRSandbox.stopConductRs(streams.value.log))
)
```


## Settings

The following settings are provided under the `SandboxKeys` object:

Name              | Scope   | Description
------------------|---------|------------
envs              | Global  | A `Map[String, String]` of environment variables to be set for each ConductR container.
image             | Global  | The Docker image to use. By default `typesafe-docker-registry-for-subscribers-only.bintray.io/conductr/conductr-dev` is used i.e. the single node version of ConductR.
imageVersion      | Global  | The version of the Docker image to use. Must be set. Please visit the [ConductR Developer](http://www.typesafe.com/product/conductr/developer) page on Typesafe.com for the current version and additional information.
ports             | Global  | A `Seq[Int]` of ports to be made public by each of the ConductR containers. This will be complemented to the `endpoints` setting's service ports declared for `sbt-bundle`.
debugPort         | Project | Debug port to be made public to the ConductR containers if the sandbox gets started in [debug mode](#Commands). The debug ports of each sbt project setting will be used. If `sbt-bundle` is enabled the JVM argument `-jvm-debug $debugPort` is  additionally added to the `startCommand` of `sbt-bundle`. Default is 5005.
logLevel          | Global  | The log level of ConductR which can be one of "debug", "warning" or "info". By default this is set to "info". You can observe ConductR's logging via the `docker logs` command. For example `docker logs -f cond-0` will follow the logs of the first ConductR container.
conductrRoles     | Global  | Sets additional roles to the ConductR nodes. Defaults to `Seq.empty`. If this settings is not set the bundle roles specified with `BundleKeys.roles` are used. To provide a custom role configuration specify a set of roles for each node. Example: Seq(Set("megaIOPS"), Set("muchMem", "myDB")) would add the role `megaIOPS` to first ConductR node. `muchMem` and `myDB` would be added to the second node. If `nrOfContainers` is larger than the size of the `conductrRoles` sequence then the specified roles are subsequently applied to the remaining nodes.

## Commands

The following commands are provided:

Name          | Description
--------------|-------------
sandbox run   | Starts the ConductR sandbox cluster without remote debugging facilities.
sandbox debug | Starts the ConductR sandbox cluster and exposes the `debugPort` to docker and ConductR to enable remote debugging.
sandbox stop  | Stops the ConductR sandbox cluster.

&copy; Typesafe Inc., 2015
