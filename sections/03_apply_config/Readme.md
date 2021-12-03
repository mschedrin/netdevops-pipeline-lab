# Applying our configuration

Now that we have created our configuration files it's time to apply them. In our process we outlined that we first want to apply them to our CML environment. To do the applying we'll use pyATS. pyATS requires a *testbed* file that specifies how to connect to our devices. While we *could* manually create this testbed file it is much easier to retrieve it programmatically from the CML. As mentioned in the setup, we'll assume that you have a CML lab called `testlab` and that the names of the devices in that lab correspond with the names of the devices in your netbox installation.

Applying our configuration is thus a two-step process

1. Retrieve the testbed file
2. Apply the configurations to those devices in the testbed where the name matches

## Retrieving the testbed file

Let's start by writing a python script that retrieves our testbed file from CML. CML was formerly known as VIRL and offers a official client library called `virl2-client`. It is included in our requirements file and should thus be installed on your system. 

Let's create a file called `get_testbed.py` in our main directory. Your directory structure should now look like this: 

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
├─ get_testbed.py
```

Our `get_testbed.py` script will take two command-line arguments. The first is the name of our lab and the second is the name of the output file. 

Let's start with some package imports

```python
from virl2_client import ClientLibrary
import sys
import yaml

if len(sys.argv) != 3:
    print("Missing arguments")
    sys.exit(-1)
```

Next, we setup our CML client instance. 

```python
# Create a client object for interacting with CML
client = ClientLibrary("https://10.10.20.161", "developer", "C1sco12345", ssl_verify=False)
```

**Note:** For this example, the connection details are hard-coded into the scripts. This is fine for a demonstration but should **never** be used in production. For production please use environment variables. You can see [here](https://docs.gitlab.com/ee/ci/variables/) how to set environment variables in gitlab.

Next, we can retrieve the lab by its name and download the pyATS testbed.

```python
# Find your lab. Method returns a list, this assumes the first lab returned is what you want
lab = client.find_labs_by_title(sys.argv[1])[0]

# Retrieve the testbed for the lab 
pyats_testbed = lab.get_pyats_testbed()
```

You might be wondering how you can connect to the simulated devices running within CML. While we could do external connectivity for each of our devices, this is cumbersome. CML offers an alternative by using the CML server itself as a jump host. This is the connection method that will be pre-configured for us in the testbed file retrieved from the CML API. However, there is one problem. The CML API will return us the credentials of the jump host, named `terminal_server` here, as `change_me` and `change_me`. So we'll have to load the YAML provided by CML, add the correct username and password, and then save the testbed file. 

```python
data = yaml.safe_load(pyats_testbed)
data['devices']['terminal_server']['credentials']['default']['username'] = "developer"
data['devices']['terminal_server']['credentials']['default']['password'] = "C1sco12345"

# Write the YAML testbed out to a file
with open(sys.argv[2], "w") as f: 
    yaml.safe_dump(data, f)
```

The same cautions regarding hard-coded credentials as mentioned above apply. 

You can run this script by using the following command:

```
$ python3 get_testbed.py testlab lab_testbed.yaml
```

You should have a `lab_testbed.yaml` file created in your directory. You can verify that this is a properly formatted testbed file by running 

```
$ pyats validate testbed lab_testbed.py
```

Or try that the connection via the jump host works by running

```
$ pyats parse "show version" --testbed-file lab_testbed.py --devices <name of your device>
```

With this done we can add the step of retrieving our testbed file to our CI/CD pipeline. 

```yaml
get_lab_testbed:
  stage: setup
  needs: ["install_dependencies"]
  script:
      - python3 get_testbed.py testlab lab_testbed.yaml
  artifacts:
    paths:
    - lab_testbed.yaml
```

Gitlab-wise there is nothing new here. We define a task called `get_lab_testbed` that needs to run after the dependencies have been retrieved. Note that this is needed here now since `install_dependencies` and `get_lab_testbed` are both part of the `setup` stage. 

We then run our script to retrieve the lab testbed and store it as an artifact. 

## Applying the configuration

Now that we have created our testbed for our virtualized testing environment we can go ahead and write a pyATS script that applies the configuration change to our devices. 

Create a file called `apply_configs.py`. Your directory structure should now look like this:

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
├─ get_testbed.py
├─ apply_configs.py
```

In this script, we are going to first load the testbed file.

```python
import sys
import glob
from genie.testbed import load

if len(sys.argv) != 3:
    print(f"Missing arguments")
    sys.exit(-1)

TESTBED = sys.argv[1]
CONFIG_DIR = sys.argv[2]
tb = load(TESTBED)
```

With our testbed loaded we can now iterate over all the configuration files in our configuration directory. This is the artifact that we saved in the `compile_templates` task of our CI/CD pipeline. 

```python
for f in glob.glob(f"{CONFIG_DIR}/*.conf"):
    device_name = f.replace(CONFIG_DIR, "").replace("/", "").replace(".conf", "")
    
    # Read config
    config = ""
    with open(f) as fh:
        config = fh.read()

    if device_name in tb.devices.keys():
        dev = tb.devices[device_name]
        dev.connect(init_exec_commands=[],
                    init_config_commands=[],
                    log_stdout=False)
        dev.configure(config)
```

In the loop, we first retrieve the device name. Remember that we named each configuration file after its device name. We then read the configuration from the file and, if the device name exists in our testbed file, connect to the device and apply the configuration commands we loaded from our file.

We can add this step to our CI/CD pipeline using the following task definition in `.gitlab-ci.yml`.

```yaml
apply_config_lab:
  stage: run
  needs: ["get_lab_testbed", "compile_templates"]
  script:
    - pyats validate testbed lab_testbed.yaml
    - python3 apply_configs.py lab_testbed.yaml configs-$CI_COMMIT_SHA
```

Notice how we now depend on multiple previous tasks (`get_lab_testbed` and `compile_templates`) and how the `apply_config_lab` task is part of a new stage called `run`. We'll thus need to add this stage to our stage list so that the top of our `.gitlab-ci.yml` file now looks like this:

```yaml
stages:
  - setup
  - compile
  - run
```

The entire `.gitlab-ci.yml` file now looks like this:

```yaml
stages:
  - setup
  - compile
  - run

install_dependencies:
  stage: setup
  script:
    - python3 -m pip install -r requirements.txt

get_lab_testbed:
  stage: setup
  needs: ["install_dependencies"]
  script:
      - python3 get_testbed.py testlab lab_testbed.yaml
  artifacts:
    paths:
    - lab_testbed.yaml

compile_templates:
  stage: compile
  needs: ["install_dependencies"]
  script:
    - mkdir -p configs-$CI_COMMIT_SHA
    - python3 compile_config.py configs-$CI_COMMIT_SHA
  artifacts:
    paths:
    - configs-$CI_COMMIT_SHA/

apply_config_lab:
  stage: run
  needs: ["get_lab_testbed", "compile_templates"]
  script:
    - pyats validate testbed lab_testbed.yaml
    - python3 apply_configs.py lab_testbed.yaml configs-$CI_COMMIT_SHA

```

Commit and push your changes to gitlab and you should see the device configuration in your lab change!

<div align="right">
   
   [Prev](../02_building_templates/Readme.md) - [Next](../04_testing/Readme.md)
</div>