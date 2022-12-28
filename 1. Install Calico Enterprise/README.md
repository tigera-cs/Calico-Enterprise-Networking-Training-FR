# In this lab

This lab provides the instructions to:

* [Install Calico Enterprise](https://github.com/Pooriya-a/CalicoEnterprise-Networking-Training/edit/main/1.%20Install%20Calico%20Enterprise/README.md)
* Deploy a three-tier sample application called "yaobank" (Yet Another Online Bank)


### Install Calico Enterprise

Upstream Kubernetes by default does not provide a network interface plugin. In this lab environment, Kubernetes has been preinstalled, but as your network plugin is not installed yet, your nodes will appear as NotReady. This section walks you through the required steps to install Calico Enteprise as both the CNI provider and network policy engine provider.

