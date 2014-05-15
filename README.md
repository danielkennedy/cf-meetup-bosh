# BOSH Lite

A local development environment for BOSH using warden containers in a Vagrant box.

This readme walks through deploying Cloud Foundry with BOSH Lite.
Bosh and BOSH Lite can be used to deploy just about anything once you've got the hang of it.

## Install

### Dependencies and Requirements

The following instructions were tested with fresh install of Ubuntu 12.04 LTS, on a machine with 8 CPUs and 16GB RAM.

```
$ sudo apt-get update
$ sudo apt-get install git bzr wget curl virtualbox golang
$ curl -L get.rvm.io | bash -s stable
$ export PATH=$PATH:$HOME/.rvm/bin
$ echo '[[ -s "$HOME/.profile" ]] && source "$HOME/.profile"' >> $HOME/.bashrc
$ echo '[[ -s "$HOME/.rvm/scripts/rvm" ]] && source "$HOME/.rvm/scripts/rvm"' >> $HOME/.bashrc
$ source ~/.rvm/scripts/rvm
$ rvm install 1.9.3
$ rvm --default use 1.9.3
$ sudo gem install bundler
$ git clone https://github.com/cloudfoundry-incubator/spiff
$ cd spiff
$ sudo go get github.com/cloudfoundry-incubator/spiff/compare
$ sudo go get github.com/cloudfoundry-incubator/spiff/flow
$ sudo go get github.com/cloudfoundry-incubator/spiff/yaml
$ sudo go get github.com/codegangsta/cli
$ sudo go get launchpad.net/goyaml
$ sudo GOOS=linux GOARCH=amd64 go build -o /usr/local/bin/spiff .
$ cd ..
$ wget https://cli.run.pivotal.io/stable?release=debian64 -O cf-cli_amd64.deb
$ sudo dpkg -i cf-cli_amd64.deb
$ wget https://dl.bintray.com/mitchellh/vagrant/vagrant_1.4.3_x86_64.deb
$ sudo dpkg -i vagrant_1.4.3_x86_64.deb
```

### Prepare the Environment

For all use cases, first prepare this project with `bundler` .

1. Run Bundler from the base directory of this repository.

    ```
    bundle
    ```

### Install and Boot a Virtual Machine

1. Start Vagrant from the base directory of this repository. This uses the Vagrantfile.

    ```
    vagrant up
    ```

1. Target the BOSH Director and login with admin/admin.

    ```
    $ bosh target 192.168.50.4
    Target set to `Bosh Lite Director'
    $ bosh login
    Your username: admin
    Enter password: *****
    Logged in as `admin'
    ```

1. Add a set of route entries to your local route table to enable direct Warden container access every time your networking gets reset (e.g. reboot or connect to a different network). Your sudo password may be required.

    ```
    scripts/add-route
    ```

## Restart the Director

Occasionally you need to restart the BOSH Lite Director to avoid https://github.com/cloudfoundry/bosh-lite/issues/82;(troubleshooting ...) so perhaps always run the following after booting up BOSH Lite:

```
vagrant ssh -c "sudo sv restart director"
```

## Upload the Warden stemcell

A stemcell is a VM template with an embedded BOSH Agent. BOSH Lite uses the Warden CPI, so we need to use the Warden Stemcell which will be the root file system for all Linux Containers created by the Warden CPI.

1. Download latest Warden stemcell

    ```
    wget http://bosh-jenkins-gems-warden.s3.amazonaws.com/stemcells/latest-bosh-stemcell-warden.tgz
    ```

1. Upload the stemcell

    ```
    bosh upload stemcell latest-bosh-stemcell-warden.tgz
    ```

NOTE: It is possible to do this in one command instead of two, but doing this in two steps avoids having to download the stemcell again when you bring up a new BOSH Lite box.

You can also use 'bosh public stemcells' to list and download the latest Warden stemcell

Example (the versions you see will be different from these):
```
$ bosh public stemcells
+---------------------------------------------+
| Name                                        |
+---------------------------------------------+
| bosh-stemcell-1722-aws-xen-ubuntu.tgz       |
| bosh-stemcell-1722-aws-xen-centos.tgz       |
| light-bosh-stemcell-1722-aws-xen-ubuntu.tgz |
| light-bosh-stemcell-1722-aws-xen-centos.tgz |
| bosh-stemcell-1722-openstack-kvm-ubuntu.tgz |
| bosh-stemcell-1722-vsphere-esxi-ubuntu.tgz  |
| bosh-stemcell-1722-vsphere-esxi-centos.tgz  |
| bosh-stemcell-24-warden-boshlite-ubuntu.tgz |
+---------------------------------------------+

$ bosh download public stemcell bosh-stemcell-24-warden-boshlite-ubuntu.tgz
```

## Deploy Cloud Foundry

1.  Install [Spiff](https://github.com/cloudfoundry-incubator/spiff). Use the [latest binary of Spiff](https://github.com/cloudfoundry-incubator/spiff/releases), extract it, and make sure that `spiff` is in your `$PATH`.

1. Clone a copy of cf-release:
    ```
	cd ~/workspace
	git clone https://github.com/cloudfoundry/cf-release
    ```

1. Decide which final release of Cloud Foundry you wish to deploy by looking at in the [releases directory of cf-release](https://github.com/cloudfoundry/cf-release/tree/master/releases).  At the time of this writing, cf-169 is the most recent. We will use that as the example, but you are free to substitute any future release.

1. Check out the desired revision of cf-release, (eg, 169)

    ````
    cd ~/workspace/cf-release
    ./update
    git checkout v169
    ````

1.  Upload final release

    Use the version that matches the tag you checked out. For v169 you would use: releases/cf-169.yml

    ```
    bosh upload release releases/cf-<version>.yml
    ```
    If the BOSH binary was not found and you use RVM, BOSH was most likely installed into the bosh-lite gemset. Switch to the gemset before uploading:

    ```
    rvm gemset use bosh-lite
    bundle
    bosh upload release releases/cf-<version>.yml
    ```

1.  Use the `make_manifest_spiff` script to create a cf manifest.  This step assumes you have cf-release checked out in ~/workspace. If you have cf-release checked out to somewhere else, you have to update the `BOSH_RELEASES_DIR`
 +environment variable.  The script also requires that cf-release is checked out with the tag matching the final release you wish to deploy so that the templates used by `make_manifest_spiff` match the code you are deploying.

    `make_manifest_spiff` will target your BOSH Lite Director, find the UUID, create a manifest stub and run spiff to generate a manifest at manifests/cf-manifest.yml. If this fails, try updating Spiff.

    ```
    cd ~/workspace/bosh-lite
    ./scripts/make_manifest_spiff
    ```

    If you want to change the jobs properties for this bosh-lite deployment, e.g. number of nats servers, you can change it in the template located under cf-release/templates/cf-infrastructure-warden.yml.


1.  Deploy CF to bosh-lite

    ```
    bosh deployment manifests/cf-manifest.yml # This will be done for you by make_manifest_spiff
    bosh deploy
    # enter yes to confirm
    ```

1.  Run the [cf-acceptance-tests](https://github.com/cloudfoundry/cf-acceptance-tests) against your new deployment to make sure it's working correctly.

    a.  Install [Go](http://golang.org/) version 1.2.1 64-bit and setup the Go environment.

    ```
    mkdir -p ~/go
    export GOPATH=~/go
    export PATH=$PATH:/usr/local/go/bin
    ```
    b.  Download the cf-acceptance-tests repository

    ```
    go get github.com/cloudfoundry/cf-acceptance-tests ...
    cd $GOPATH/src/github.com/cloudfoundry/cf-acceptance-tests
    ```
    c.  Follow the [cats](https://github.com/cloudfoundry/cf-acceptance-tests) instructions on Running the tests.


### Single command deploy

Alternatively to the above steps, you can also run this script to deploy the latest version of CloudFoundry:

```
$ ./scripts/provision_cf
```

## Try your Cloud Foundry deployment

Install the [Cloud Foundry CLI](https://github.com/cloudfoundry/cli) and run the following:

```
cf api http://api.10.244.0.34.xip.io
cf auth admin admin
cf create-org me
cf target -o me
cf create-space development
cf target -s development
```
	
Now you are ready to run commands such as `cf push`.	

## SSH into deployment jobs

Use `bosh ssh` to SSH into running jobs of a deployment and run the following command:

```
scripts/add-route
```

Now you can now SSH into any VM with `bosh ssh`:

```
$ bosh ssh
1. nats/0
2. syslog_aggregator/0
3. postgres/0
4. uaa/0
5. login/0
6. cloud_controller/0
7. loggregator/0
8. loggregator-router/0
9. health_manager/0
10. dea_next/0
11. router/0
Choose an instance:
```

## Restore your deployment

The Warden container will be lost after a vm reboot, but you can restore your deployment with `bosh cck`, BOSH's command for recovering from unexpected errors.
```
$ bosh cck
```

Choose `2` to recreate each missing VM:
```
Problem 1 of 13: VM with cloud ID `vm-74d58924-7710-4094-86f2-2f38ff47bb9a' missing.
  1. Ignore problem
  2. Recreate VM using last known apply spec
  3. Delete VM reference (DANGEROUS!)
Please choose a resolution [1 - 3]: 2
...
```
Type yes to confirm at the end:
```
Apply resolutions? (type 'yes' to continue): yes

Applying problem resolutions
  missing_vm 212: Recreate VM using last known apply spec (00:00:13)
  missing_vm 215: Recreate VM using last known apply spec (00:00:08)
...
Done                    13/13 00:03:48
```

## Troubleshooting

1. Starting over again is often the quickest path to success; you can use `vagrant destroy` from the base directory of this project to remove the VM.
1. To start with a new VM, just execute the appropriate `vagrant up` command, optionally passing in the provider as shown in the earlier sections.

## Manage your local boxes

We publish pre-built Vagrant boxes on Amazon S3. It is recommended to use the latest boxes.

### Download latest boxes

Just get a latest copy of the Vagrantfile from this repo and run `vagrant up`.

## Working offline

See the [offline documentation](docs/offline_dns.md)
