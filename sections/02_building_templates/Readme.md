# Building Templates

Our first step is to build a script that can create a device configuration template. We have already looked at this previously during our session on 
building a single source of truth. The recording is available on our [salesconnect page](https://salesconnect.cisco.com/#/program/PAGE-15177).

Let's start by creating a folder called `templates` that will store our configuration templates.

Next, we need to create a python script called `compile_config.py`. This script can be called manually during the process of building your configuration templates to get a look at what is being generated as well as being used by the CI/CD pipeline to generate your configuration. 

First, we'll need some utility functions. Create a file called `utils.py` with the 
following function:

```python
def get_devices(netbox, tag):
    return netbox.dcim.get_devices(tag=tag)
```

This short function retrieves all devices registered to netbox associated with a specific tag from your netbox. In this pipeline we'll be associating `tags` with `configuration templates`. That means that, based on the tag of a device, the corresponding template will be used. To feed this information into our system we need a mapping. Create a file called `mapping.json` with the following content:

```json
{
    "router": "router.tpl.conf"
}
```

Note that this assumes that the tag that you have assigned in netbox is called *router*.

Your directory structure should look like this:

```
netdevops-cicd-test/
├─ templates/
├─ mapping.json
├─ utils.py
├─ compile_config.py
```
The final step before being able to write our compile script is to add our config templates. Please refer to the session on building a single source of truth for more in-depth information on the particulars of building a device configuration template with jinja2.

In your templates folder, create a subfolder called `components`. We'll split our configuration template into reusable components (for example one component for specifying general device configuration information such as a message of the day) that will be pulled in by our router configuration. 

Inside of the components folder, create a file called `motd.tpl.conf` with the following content

```
banner motd $ {{ motd }} $
```

As you can see, this is just a templated version of a configuration command to set the message of the day of a cisco device. 

We'll now create the template for our router configuration that will consume the component we just created. In your `templates` folder, create a file called `router.tpl.conf`. 

Note that the name of your configuration needs to match the name of the configuration file you have associated with a tag in your `mapping.json` file.

Inside of the `router.tpl.conf` file we'll simply import our component for now.

```
{% include 'components/motd.tpl.conf' %}
```

Your directory structure should now look like this:

```
netdevops-cicd-test/
├─ templates/
│  ├─ router.tpl.conf
│  ├─ components/
│  │  ├─ motd.tpl.conf
├─ mapping.json
├─ utils.py
├─ compile_config.py
```

With the preparations done we can open up our `compile_config.py` file.

We start by importing our libraries:

```python
import json
import sys
import os 

import jinja2

from netbox import NetBox
from pathlib import Path

from utils import get_devices
```

Next, we setup our connection to netbox and create a jinja2 environment.

```python
netbox = NetBox(host='127.0.0.1', port=8000, use_ssl=False, auth_token='0123456789abcdef0123456789abcdef01234567')
loader = jinja2.FileSystemLoader(searchpath="templates")
env = jinja2.Environment(loader=loader)
```

**Note:** For this example, the connection details are hard-coded into the scripts. This is fine for a demonstration but should **never** be used in production. For production please use environment variables. You can see [here](https://docs.gitlab.com/ee/ci/variables/) how to set environment variables in gitlab.

Next, we retrieve the target directory from our command-line arguments, make sure it exists, and load our mappings. 

```python
TARGET_DIR = sys.argv[1]
Path(TARGET_DIR).mkdir(parents=True, exist_ok=True)

conf_mapping = json.load(open("mapping.json"))
```

With this setup done we can now loop over all our tags that are known in our mapping, retrieve all devices from netbox that have that tag, load the template associated with the tag and then compile a template for each of the devices in that tag. To do so, we retrieve the device specific `config_context` from netbox and pass it into jinja2. Finally, we write the newly created config to our `TARGET_DIR` folder using the device name as the name of our configuration file. 

```python
for tag, template in conf_mapping.items():
    devices = get_devices(netbox, tag)

    # Load template
    tpl = env.get_template(template) 

    # Compile a config for each device
    for dev in devices:
        # Create context
        ctx = {}
        for k, v in dev['config_context'].items():
            ctx[k] = v

        out = tpl.render(**ctx)        
        
        target_file = os.path.join(TARGET_DIR, f"{dev['name']}.conf")
        with open(target_file, "w") as f:
            f.write(out)
```

You can now try and run this script by using the following command:

```
$ python3 compile_config.py TEST
```

This will create a folder called `TEST` in your main directory and in it you should find a compiled template using the configuration context that you specified in Netbox.

Once we have verified that our script works we can have it run in our CI/CD pipeline. Create a file called `.gitlab-ci.yml` in the main directory of your repository. Your directory structure should now look like this:

```
netdevops-cicd-test/
├─ templates/
│  ├─ router.tpl.conf
│  ├─ components/
│  │  ├─ motd.tpl.conf
├─ mapping.json
├─ utils.py
├─ compile_config.py
├─ .gitlab-ci.yml
```

In this file, we'll create the instructions for gitlab what to do in our CI/CD pipeline. 

Start by defining two stages: 
* In the `setup` stage we'll install our requirements
* In the `compile` stage we'll then compile our configuration templates

```yaml
stages:
    - setup
    - compile
```

Next, we can define tasks that should be run within that stage. Define a task to install our dependencies and associate it with the setup stage:

```yaml
install_dependencies:
  stage: setup
  script:
    - python3 -m pip install -r requirements.txt
```

While stages run in sequence (i.e. all tasks in the `compile` stage will run *after* all tasks from the `setup` stage have been completed), we can also explicitly create a dependency. This is redundant here since the `compile_templates` task is in a stage that runs *after* the `setup` stage and thus after the `install_dependencies` task but it is nonetheless a nice thing to know. The dependency is specified using the `needs` directive. 

```yaml
compile_templates:
  stage: compile
  needs: ["install_dependencies"]
  script:
    - mkdir -p configs-$CI_COMMIT_SHA
    - python3 compile_config.py configs-$CI_COMMIT_SHA
```

Notice how we are using a environment variable called `CI_COMMIT_SHA`. This environment variable is set by gitlab and will contain the commit sha of the commit that is currently being compiled. This allows us to create a unique directory name for each of our commits. 

Since we'll need these configuration files in our next steps we'll have to define the folder as an artifact. 

```yaml
artifacts:
    paths:
    - configs-$CI_COMMIT_SHA/
```

Your finale `.gitlab-ci.yml` file should look like this:

```yaml
stages:
    - setup
    - compile

install_dependencies:
  stage: setup
  script:
    - python3 -m pip install -r requirements.txt

compile_templates:
  stage: compile
  needs: ["install_dependencies"]
  script:
    - mkdir -p configs-$CI_COMMIT_SHA
    - python3 compile_config.py configs-$CI_COMMIT_SHA
  artifacts:
    paths:
    - configs-$CI_COMMIT_SHA/ 
```

Commit all the files in this repository and push them - this should trigger the pipeline to kick off and your configuration should be build. 

<div align="right">
   
   [Prev](../01_setup/Readme.md) - [Next](../03_apply_config/Readme.md)
</div>