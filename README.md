# Vault Upgrade Hook

The *Vault-Upgrade-Hook* is an easy way to add additional logic to the installation of content packages. Nearly all projects are facing sooner or later the problem that existing user generated content has to be upgraded. Often the existing mechanism with plain content packages, filters and install modes is not enough and more complex and flexible approaches are needed. The *Vault-Upgrade-Hook* provides an alternative to implementing new Jars for every of those jobs by embedding additional actions (i.e. groovy scripts) directly into the content package. 

## Feature Overview

- installation mode "always" to execute on every package installation 
- installation mode "onChange" to execute only new and changed actions
- minimum dependencies
- flexible API for custom action functionality
- convention over configuration, but still many options

## Requirements
 
`AEM6.0 SP3` and above, if you are using Sling without AEM see the detailed dependencies in `vault-upgrade-hook/pom.xml`.

## Usage

Two general ways of how to use it: 

1. ad-hoc admin style:
    1. take a copy of one of the prepared projects under `samples/...`, 
    1. update actions under `src/main/upgrader` (e.g. test-groovy - replace all scripts with yours),
    1. build and install: `mvn -Pinstall`, target server and credentials can be set via `-Dcrx....`

2. dev-style:
    1. integrate the JAR via maven copy to your content package:
        ```
        <plugin>
          <groupId>org.apache.maven.plugins</groupId>
          <artifactId>maven-dependency-plugin</artifactId>
          <executions>
            <execution>
              <id>copy</id>
              <phase>prepare-package</phase>
              <goals>
                <goal>copy</goal>
              </goals>
              <configuration>
                <artifactItems>
                    <artifactItem>
                        <groupId>biz.netcentric.vlt.upgrade</groupId>
                        <artifactId>vault-upgrade-hook</artifactId>
                        <version>${vault.upgrade.hook.version}</version>
                        <overWrite>true</overWrite>
                        <outputDirectory>${project.build.directory}/vault-work/META-INF/vault/hooks</outputDirectory>
                    </artifactItem>
                </artifactItems>
              </configuration>
            </execution>
          </executions>
        </plugin>
        ```
    2. create a directory (like `samples/groovy-package/src/main/upgrader/test-groovy`) and place it in your content package under `META-INF/vault/definition/upgrader`.

Note that the general structure of the package is always the same. There is a folder `META-INF/vault/definition/upgrader/<upgrade-info>` in the content package which contains the configuration properties for the contained actions. Depending on the used handler those actions are child nodes containing Groovy scripts or SlingPipes.   

## More information

### Jackrabbit hook

AEM/Jackrabbit content packages allow to place JARs to a package in `META-INF/vault/hooks` those must contain a class implementing `InstallHook` and will be executed during installation. The installation is split into phases: `PREPARE`, `INSTALLED` and `END`. `END` is only called if installation was successful otherwise `PREPARE_FAILED` or `INSTALL_FAILED` are called on errors. The *Vault-Upgrade-Hook* uses this mechanism and builds an upgrade process on top of it.

### Upgrade Process

The upgrade process is embedded in the installation phases of the package. On `PREPARE` the status of the last execution will be loaded and on `END` the status will be saved. On all phases `UpgradeAction`s will be executed if the package contains actions for the current phase. `UpgradeAction`s are bundled in so called `UpgradeInfo`s which provide general configuration how the actions should be executed. For example does the `UpgradeInfo`s define to use Groovy scripts for the upgrade or SlingPipes. Also options like the `mode` which decides about whether to execute actions will be configured there.

Digging a level deeper in the implementation the process is as follows: on installation of the content package `biz.netcentric.vlt.upgrade.UpgradeProcessor.execute(InstallContext)` will be called for each of the phases ([https://jackrabbit.apache.org/filevault/apidocs/org/apache/jackrabbit/vault/packaging/InstallContext.Phase.html]). The processor will read the status of previous executions from `/var/upgrade` and loads the `biz.netcentric.vlt.upgrade.UpgradeInfo` child nodes from the current content package under `<package-path>/jcr:content/vlt:definition/upgrader`. On `END` the the list of all executed actions is stored to `/var/upgrade`.

An `UpgradeInfo` loads a `biz.netcentric.vlt.upgrade.handler.UpgradeHandler` implementation to create `biz.netcentric.vlt.upgrade.handler.UpgradeAction`s which are executed during the upgrade. Whether an `UpgradeInfo` and an `UpgradeAction` is executed depends on some attributes:

- if the `installationMode` is not set explicitly or set to `OnChange` (default) only new and changed actions are executed 
- if the `installationMode` is explicitly set to "Always" 

This behaviour can be changed by configuration options 
- `installationMode="always"` - actions of this info will always be executed disregarding of previous upgrades
- `skipOnInitial="false"` - actions will also be executed if it is the first execution of the upgrade

`UpgradeAction`s are bound to a specific execution phase. The default Phase is `INSTALLED`. This means an arbitrary action is executed after the content got installed. This can be overridden by prefixing the groovy script name with the name of another phase e.g. "prepare_failed-myscript.groovy".

### Upgrade Actions
Multiple different upgrade actions are included with this hook. Those are also referred to as handlers. For details refer to the following sections.

#### Groovy

For usage and details please see the [sample package](samples/groovy-package).

#### Sling Pipes

For details about Sling Pipes please have a look at [Sling documentation](https://sling.apache.org/documentation/bundles/sling-pipes.html) and the [sample package](samples/sling-pipes-package).

#### User Preferences

For usage and details please see the [sample package](samples/userpreferences-package).

### Configuration

For a full list of configuration options and their descriptions please see the JavaDocs of biz.netcentric.vlt.upgrade.UpgradeInfo.
