# Testing our Configuration

The final step of our pipeline is to run the tests. The session on *advanced pyATS* explained in more depth how to write a pyATS test case. 

The brief version is that each test case consists of

* a `CommonSetup` in which common tasks such as connecting to the device is carried out
* one of more `Testcase` instances in which the actual testing (i.e. verifying the version of a operating system) is carried out
* a `CommonCleanup` where cleanup tasks such as disconnecting from the devices are carried out once all test cases have run.

Create a file called `test_version.py` in your main directory. Your directory structure should now look like this:

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
├─ test_version.py
```

In our `test_version.py` file, import the required packages and define our common setup. 

```python
# import the aetest module
from pyats import aetest
import logging
import os

logger = logging.getLogger(__name__)

class CommonSetup(aetest.CommonSetup):
    
    @aetest.subsection
    def connect_to_devices(self, testbed):
        
        for device in testbed:
            # don't do the default show version
            # don't do the default config
            device.connect(init_exec_commands=[],
                           init_config_commands=[],
                           log_stdout=False)
            
            logger.info('{device} connected'.format(device=device.alias))
```

As you can see, we are simply using pyATS and the passed testbed file to connect to all our devices.

Next, let's define our test case. 
```python
class CheckVersion(aetest.Testcase):

    @aetest.test
    def check_current_version(self, testbed):
        # Local vars
        test_success = True
        xe_version = '15.9(3)M3'

        for device in testbed:
            if device.alias == "terminal_server":
                continue

            #Learning platform information
            platform = device.learn('platform')
            # We use .lower() so we're not case sensitive
            if (platform.os.lower() == 'ios' and platform.version != xe_version): 
                test_success = False
                logger.error(f"{device.alias} is {platform.version} - should be {xe_version}")


            logger.debug('{device} running {platformos} has OS: {os}'.format(device=device.alias, platformos=platform.os, os=platform.version))
        
        assert test_success == True
```

In this test case we iterate over all devices in our testbed to then learn the platform features and check if the version number is correct. Note that, due to the way that we are connecting to our devices via the `terminal_server` jumphost, that jumphost is part of our testbed and thus pyATS would try and learn the platform of the CML server. Since this is not available we need to explicitly skip it. 

With our testcase defined we just need to write our cleanup tasks in which we disconnect from all devices and also specify how to run our test cases when the script is invoked. 

```python
class CommonCleanup(aetest.CommonCleanup):

    @aetest.subsection
    def disconnect_from_devices(self, testbed):
        
        for device in testbed:
            device.disconnect()

if __name__ == '__main__':

    # local imports
    from genie.testbed import load
    import sys

    # set debug level DEBUG, INFO, WARNING
    logger.setLevel(logging.INFO)

    # Loading device information
    testbed = load(sys.argv[1])

    aetest.main(testbed = testbed)
```

You can run your test locally by using the following command:

```
$ python3 test_version.py lab_testbed.yaml
```

Finally, we can create a new stage called `test` in our gitlab pipeline and run this script using the task definition below. 

```yaml
run_tests:
  stage: test
  needs: ["apply_config_lab", "get_lab_testbed"]
  script:
    - python3 test_version.py lab_testbed.yaml
```

Our final `.gitlab-ci.yml` file thus looks like this:

```yaml
stages:
  - setup
  - compile
  - run
  - test

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

run_tests:
  stage: test
  needs: ["apply_config_lab", "get_lab_testbed"]
  script:
    - python3 test_version.py lab_testbed.yaml
```

Commit and push this to your repository and you have succesfully compiled configuration tempaltes, applied the configuration to a testing environemnt and then run a pyATS test against those devices - all from a CI/CD pipeline. 