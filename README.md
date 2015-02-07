# sbt-deploy-ssh
SBT deploy plugin

Allows you to setup deploy configuration for your project.

You will be able to deploy your project with `deploy-ssh` task

Usage example `deploy-ssh yourServerName1 yourServerName2 ...`

**autoplugin (sbt >= 0.13.5)**

## Current version: 0.1.1

[Please read sbt documentation before start to work with plugin](http://www.scala-sbt.org/0.13.5/docs/Getting-Started/Using-Plugins.html)

 - [Installation](#installation)
 - [Configuration](#configuration)
  - [Configs](#configs)
  - [Locations](#locations)
  - [Artifacts](#artifacts)
 - [Execute scripts before/after deploy](#execute-scripts-beforeafter-deploy) 
 - [Link to task](#link-to-task)

## Installation

Add to your project/plugins.sbt file:

``` sbt
addSbtPlugin("com.github.shmishleniy" % "sbt-deploy-ssh" % "0.1.1")
```

Add ssh library repository in SBT

```sbt
resolvers += "JAnalyse Repository" at "http://www.janalyse.fr/repository/"
```

Enable plugin in your project.
For example in your `build.sbt`

``` sbt
lazy val myProject = project.enablePlugins(DeploySSH)
```

## Configuration

### Configs

You can specify configs that will be used for deployment.

You can use `.conf` files or set configs directly in project settings.

Allowed config fields:

* `name` - your server name. **Should be unique** in all loaded configs. (Duplication will be overriden)
* `host` - ip adress or hostname of the server
* `user` - ssh username. If missing or empty will be used your current user (`user.name`)
* `password`- ssh password. If missing or empty will be used ssh key
* `passphrase`- passphrase for ssh key. Remove or leave empty for ssh key without passphrase
* `port` - ssh port. If missing or empty will be used `22`
* `sshDir` - directory with you ssh keys. This directory should contain `id_rsa` or `id_dsa`. By default `user.name/.ssh` directory. This field is not allowed to be empty in `.conf` file. You should remove this from config in `.conf` file to use default value.

**`name` and `host` fields are mandatory**

To set server configs in project settings use `ServerConfig` class and `deployConfigs` task key (see details below in `Locations` section)

``` scala
case class ServerConfig(name: String,
                        host: String,
                        user: Option[String] = None,
                        password: Option[String] = None,
                        passphrase: Option[String] = None,
                        port: Option[Int] = None,
                        sshDir: Option[String] = None)
````

Example of the `.conf`
``` conf
servers = [
 #connect to the server via `22` port and ssh key that located in `user.name/.ssh/` directory, user is current `user.name`
 {
  name = "server_0"
  host = "127.0.0.1"
 },
 #connect to the server via `22` port and ssh key that located in `/tmp/.sshKeys/` directory, user is `ssh_test`
 {
  name = "server_1"
  host = "169.254.0.2"
  user = "ssh_test"
  sshDir = "/tmp/.sshKeys"
 }
]
```

### Locations
There are four places where you can store your server config (All configs will be loaded and merged).

* Extrenal config file that located somewhere on your PC
* Config file located in your project directory
* Config file located in user home directory
* Set server configs directly in project settings

``` sbt 
lazy val myProject = project.enablePlugins(DeploySSH).settings(
 //load build.conf from external path
 deployExternalConfigFile ++= Seq("/home/myUser/Documents/build.conf"),
 //load build2.conf from `myProjectDir` and load build3.conf from `myProjectDir/projects`
 deployResourceConfigFile ++= Seq("build2.conf", "projects/build3.conf"),
 //load build4.conf from user home directory (in example `/home/myUser/build4.conf`)
 deployHomeConfigFile ++= Seq("build4.conf"),
 //configuration in project setttings
 deployConfigs ++= mySettings,
 deployConfigs ++= Seq(
  ServerConfig("server_6", "169.254.0.3"),
  ServerConfig("server_7", "169.254.0.4")
 )
)

val mySettings = Seq(
 ServerConfig("server_5", "169.254.0.2")
)
```

### Artifacts
Set artifacts to deploy

``` sbt 
lazy val myProject = project.enablePlugins(DeploySSH).settings(
 version := "1.1",
 deployConfigs ++= Seq(
  ServerConfig("server_5", "169.254.0.2")
 ),
 deployArtifacts ++= Seq(
  //`jar` file from `packageBin in Compile` task will be deployed to `/tmp/` directory
  ArtifactSSH((packageBin in Compile).value, "/tmp/"),
  //directory `stage` generated by `sbt-native-packager` will be deployed to `~/stage_1.1_release/` directory
  ArtifactSSH((stage in Universal).value), s"stage_${version.value}_release/")
 )
)
```

Deploy execution for this config:

`deploy-ssh server_5`

### Execute scripts before/after deploy

Use `deploySshExecBefore` and `deploySshExecAfter` to execute any bash commands before and after deploy.

Any exeption in `deploySshExecBefore` and `deploySshExecAfter` will abort deploy for all servers.

To skip deploy only for curent server you should wrap exeption to `SkipDeployException`.

``` sbt
lazy val myProject = project.enablePlugins(DeploySSH).settings(
 deployConfigs ++= Seq(
  ServerConfig("server_5", "169.254.0.2")
 ),
 deploySshExecBefore ++= Seq(
  (ssh: SSH) => {
   ssh.execOnce("touch pid")
   val pid = ssh.execOnceAndTrim("cat pid")
   if ("".equals(pid)) {
    //skip deploy to current server
    throw new SkipDeployException(new RuntimeException("missing pid"))
   }
   ssh.execOnceAndTrim(s"kill $pid")
  }
 ),
 deploySshExecAfter ++= Seq(
  (ssh: SSH) => {
   ssh.execOnce("nohup ./myApp/run & echo $! > ~/pid")
   ssh.execOnce("touch pid")
   val pid = ssh.execOnceAndTrim("cat pid")
   if ("".equals(pid)) {
    //stop deploy to all servers
    throw new RuntimeException("missing pid. please check package")
   }
  }
 )
)
```

### Link to task

If you need execute deploy in your task you can use `deploySshTask` and `deploySshServersNames` to config args for `deploySsh`. Or cast `deploySsh` to task. 

``` sbt
lazy val myProject = project.enablePlugins(DeploySSH).settings(
 deployConfigs ++= Seq(
  ServerConfig("server_5", "169.254.0.2"),
  ServerConfig("server_6", "169.254.0.3")
 ),
 deploySshServersNames ++= Seq("server_5", "server_6"),
 publishLocal := deploySshTask //or deploySsh.toTask(" server_5 server_6")
)
```
