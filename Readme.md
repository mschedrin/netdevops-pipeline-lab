# PIW FY22 - Pipeline Lab Guide

This lab guide walks you through the steps taken during our practical 
NetDevOps pipeline example. In this guide we'll be setting up a NetDevOps pipeline to compile configuration templates written in jinja2 using 
information provided by our DCIM/IPAM system. 

We then push these configurations to a virtualized lab. In this virtualized lab we then run a testcase against our lab equipment before finishing our pipeline. 

In this example we are going to use the following tools:

* [netbox](https://netbox.readthedocs.io/en/stable/) as our DCIM/IPAM system
* [gitlab](gitlab.com/) as our git hosting and CI/CD runner system
* [CML](https://developer.cisco.com/docs/modeling-labs/) for virtualizing our lab environment
* [pyATS](https://pubhub.devnetcloud.com/media/pyats/docs/) for writing and running our test cases

The next section will give a brief overview of how to setup these components. This guide is meant to be consumed together with the recording. Please refer to the session for more in-depth explanations.

<div align="right">
   
   Prev - [Next](sections/01_setup/Readme.md)
</div>
