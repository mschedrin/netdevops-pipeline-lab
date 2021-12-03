# Setup and Prerequisists

In order to use this system you'll require a few things.

1. Setup a gitlab account on [gitlab.com](https://gitlab.com). We will push our configuration changes to a repository on this hosting provider. Go ahead and create a repository that will host our configuration.
2. Register a gitlab runner. The gitlab runner is a piece of software that can be registered to your gitlab instance and runs your pipeline everytime a new commit comes in. This runner needs to have reachability into CML, netbox and gitlab. Assuming that you are using the DevNet Sandboxes for trying this out it is easiest if you have the gitlab runner setup on your local machine. Follow the installation instructions [here](https://docs.gitlab.com/runner/install/index.html) and [associate it with your gitlab instance](https://docs.gitlab.com/runner/register/). For the execution engine you'll want to use *Shell* for this example. The machine the runner is running on *needs* to be able to reach both the CML instance as well as your netbox installation.
3. Setup Netbox. The easiest way to do this is to follow the [netbox docker quickstart guide](https://github.com/netbox-community/netbox-docker). 
4. Get access to CML. The easiest way to do this is by using one of the DevNet sandboxes. You can find and register the CML sandbox by navigating [here](https://devnetsandbox.cisco.com/RM/Diagram/Index/45100600-b413-4471-b28e-b014eb824555?diagramType=Topology).

You should now have:

* A repository created on gitlab.com
* A gitlab runner associated with your account on gitlab.com
* Access to CML
* Access to a Netbox instance

Additionally, you'll need to install some python dependencies. You can use the following `requirements.txt` file and install it by running `python3 -m pip install -r requirements.txt`. Place this file in the root directory of your repository so that the dependencies can also be installed in the pipeline.

```
python-netbox
virl2-client
pyats
pyyaml
jinja2
```

You'll need a lab with at least one device created in your CML environment. Pleaes refer to the the session on CML for more in-depth instructions on how to do this. In your netbox, you'll want to create a device that matches the name of your CML testing environment. Please refer to the session recording or the recording on setting up a single source of truth for your network for information on how to do this. 

<div align="right">
   
   [Prev](../../Readme.md) - [Next](../02_building_templates/Readme.md)
</div>