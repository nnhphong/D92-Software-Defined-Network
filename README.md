This project comprises a set of excercises from [ONF](https://docs.google.com/presentation/d/162MCrgvxhTp-IYynZJHkDVrNziNLb8DDJXPYfqGLbzI/edit?slide=id.g77f303cb0b_0_1053#slide=id.g77f303cb0b_0_1053) 
with the end goal is building a leaf-spine data center fabric based on IPv6, using P4, Stratum, and ONOS. With the building blocks of next-generation SDN (NG-SDN) architecture, such as:

* Data plane programming and control via P4 and P4Runtime
* Configuration via YANG, OpenConfig, and gNMI
* Stratum switch OS
* ONOS SDN controller

## System requirements  
### Option 1 - Download tutorial VM

Use the following link to download the VM (4 GB):
* <http://bit.ly/ngsdn-tutorial-ova>

The VM is in .ova format and has been created using VirtualBox v5.2.32. To run
the VM you can use any modern virtualization system, although we recommend using
VirtualBox. For instructions on how to get VirtualBox and import the VM, use the
following links:

* <https://www.virtualbox.org/wiki/Downloads>
* <https://docs.oracle.com/cd/E26217_01/E26796/html/qs-import-vm.html>

Alternatively, you can use the scripts in [util/vm](util/vm) to build a VM on
your machine using Vagrant.

**Recommended VM configuration:**
The current configuration of the VM is 4 GB of RAM and 4 core CPU. These are the
recommended minimum system requirements to complete the exercises. When
imported, the VM takes approx. 8 GB of HDD space. For a smooth experience, we
recommend running the VM on a host system that has at least the double of
resources.

**VM user credentials:**
Use credentials `sdn`/`rocks` to log in the Ubuntu system.

### Option 2 - Manually install Docker and other dependencies

All exercises can be executed by installing the following dependencies:

* Docker v1.13.0+ (with docker-compose)
* make
* Python 3
* Bash-like Unix shell
* Wireshark (optional)

## Download / upgrade dependencies

The VM may have shipped with an older version of the dependencies than we would
like to use for the exercises. You can upgrade to the latest version using the
following command:

    cd ~/ngsdn-tutorial
    make deps

This command will download all necessary Docker images (~1.5 GB) allowing you to
work off-line. For this reason, we recommend running this step ahead of the
tutorial, with a reliable Internet connection.

## Repo structure

This repo is structured as follows:

 * `p4src/` P4 implementation
 * `yang/` Yang model used in exercise 2
 * `app/` custom ONOS app Java implementation
 * `mininet/` Mininet script to emulate a 2x2 leaf-spine fabric topology of
   `stratum_bmv2` devices
 * `util/` Utility scripts
 * `ptf/` P4 data plane unit tests based on Packet Test Framework (PTF)

## Commands

To facilitate working on the exercises, we provide a set of make-based commands
to control the different aspects of the tutorial. Commands will be introduced in
the exercises, here's a quick reference:

| Make command        | Description                                            |
|---------------------|------------------------------------------------------- |
| `make deps`         | Pull and build all required dependencies               |
| `make p4-build`     | Build P4 program                                       |
| `make p4-test`      | Run PTF tests                                          |
| `make start`        | Start Mininet and ONOS containers                      |
| `make stop`         | Stop all containers                                    |
| `make restart`      | Restart containers clearing any previous state         |
| `make onos-cli`     | Access the ONOS CLI (password: `rocks`, Ctrl-D to exit)|
| `make onos-log`     | Show the ONOS log                                      |
| `make mn-cli`       | Access the Mininet CLI (Ctrl-D to exit)                |
| `make mn-log`       | Show the Mininet log (i.e., the CLI output)            |
| `make app-build`    | Build custom ONOS app                                  |
| `make app-reload`   | Install and activate the ONOS app                      |
| `make netcfg`       | Push netcfg.json file (network config) to ONOS         |
