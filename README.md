# sbt-deploy-ssh
SBT deploy plugin

Allows you to setup deploy configuration for your project.

Usage example `sbt deploy-ssh yourServerName`

**autoplugin (sbt >= 0.13.5)**

[Please read sbt documentation before start to work with plugin](http://www.scala-sbt.org/0.13.5/docs/Getting-Started/Using-Plugins.html)

- [Usage](#usage)
  - [Installation](#installation)
  - [Configuration](#configuration)
    - [Configs](#configs)
    - [Locations](#locations)

## Usage

### Installation

Add to your project/plugins.sbt file:
``` sbt
addSbtPlugin("com.github.shmishleniy" % "sbt-deploy-ssh" % "0.1")
```
Enable plugin in your project.
For example in your `build.sbt`
``` sbt
lazy val myProject = project.enablePlugins(DeploySSH)
```

### Configuration

#### Configs

You can specify configs that will be used for deployment.

You can use `.conf` files or set configs directly in project settings.

Allowed config fields:

* `name` - your server name. **Should be unique** in all loaded configs. (Duplication will be overriden)
* `host` - ip adress or hostname of the server
* `user` - ssh username. If missing or empty will be used your current use (`user.name`)
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

#### Locations
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
  deployConfigs ++= Seq(
    ServerConfig("server_5", "127.0.0.1")
  )
)
```