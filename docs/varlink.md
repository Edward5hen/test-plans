# libvarlink and python-varlink packages test plan

## 1. Test Plan Identifier
This test plan will be used to test packages **libvarlink** and **python-varlink** which are for RHEL8. For us, these 2 packages are used for tool `podman` with the feature `podman varlink`, and this feaure will eventually be needed for OpenStack team.

!!! note
    In this test plan we mainly test the parts related to `podman varlink`. For `libvarlink`, we only test command tool `varlink`, which can be used after install `libvarlink-util` package.

## 2. Introduction:

> Erratas Advisories
>
> * [https://errata.devel.redhat.com/advisory/37628](https://errata.devel.redhat.com/advisory/37628)
>
> * [https://errata.devel.redhat.com/advisory/37688](https://errata.devel.redhat.com/advisory/37688)

[Varlink](https://varlink.org) is an interface description format and protocol that aims to make services accessible to both humans and machines in the simplest feasible way. Varlink is plain-text, type-safe, discoverable, self-documenting, remotable, testable, easy to debug.

* [libvarlink](https://github.com/varlink/libvarlink) is the C implementation Of `varlink`. The `libvarlink-util` package provides the command line tool `varlink`.

* [python-varlink](https://github.com/varlink/python) is the python varlink implementation. Users can install from `pypi` with `pip`, or directly from `yum/dnf`.


## 3. Test Items[^1]

[^1]: Tests scripts locates in https://github.com/Edward5hen/varlink-test

### 3.1 Installation Test
Test the packages can be succesfully installed.

```Bash tab="libvarlink"
rpm -ivh libvarlink-xxxxx.rpm
rpm -ivh libvarlink-util-xxxxxx.rpm
```

```Bash tab="python-varlink"
rpm -ivh python-varlink-xxxx.rpm
```

!!! success
    Exit code shoud be 0

### 3.2 Tests from upstream
This is only available for `python-varlink` now, test the [examples](https://github.com/varlink/python#examples) upstream has.

### 3.3 Podman Varlink Methods and Errors Test

!!! note "Prerequisites"
    1. `podman` should be installed on host
    2. `io.podman.socket` should be enabled and started, which can refer to [DOC](https://github.com/containers/libpod/blob/master/docs/podman-varlink.1.md)

`podman` provides the varlink interface `io.podman`. There are a lot of *methods* supported. This test plan is not going to cover all the methods,  but some important and most used ones in a container's **lifecycle**: `PullImage()`, `ListImages()`, `CreateContainer()`, `GetContainer()`, `ListContainers()`, `StartContainer()`, `PauseContainer()`, `UnpauseContainer()`, `StopContainer()`, `RemoveContainer()`, `RemoveImage()`. As well as the errors: `ContainerNotFound()`, `ImageNotFound()`.

<!---
```sequence
Title: methods tested
None->Image: PullImage(), ListImages()
Image->Created container: CreateContainer(), GetContainer(), ListContainers()
Created container->Running container: StartContainer()
Running container->Paused Container: PauseContainer()
Paused Container->Running Container: UnpauseContainer()
Running container->Stopped Container: StopContainer()
Stopped Container->None: RemoveContainer(), RemoveImage()
```
-->

!!! info
    All the types, methods and errors `podman` supports can refer to [this file](https://github.com/containers/libpod/blob/master/cmd/podman/varlink/io.podman.varlink).

Below are the test scripts:

```Bash tab="libvarlink"
#!/bin/bash
#
# ----------------------------------------------------------------
# This script is to test varlink command works well with podman.
#
# Prerequisites:
#     1. Host must be RHEL8.
#     2. Yum module container-tools is installed.
#
# Test steps:
#     1. Pull image. -->> varlink method: PullImage()
#     2. Check image is pulled. -->> varlink method: ListImages(), GetImage()
#     3. Run image. -->> varlink method: CreateContainer(), GetContainer()
#     4. Pause container. -->> varlink method: PauseContainer()
#     5. Unpause container. -->> varlink method: UnpauseContainer()
#     6. Stop container. -->> varlink method: StopContainer()
#     7. Remove container. -->> valink method: RemoveContainer()
#     8. Get non-exist image. -->> varlink error: ImageNotFound
#     9. Get non-exist container. -->> varlink error: ContainerNotFound
# ---------------------------------------------------------------

# ------------------------ Global Variables -------------------------
CMD_PREFIX="varlink call unix:/run/podman/io.podman/io.podman."

# ------------------------ Functions --------------------------------
verify() {
    if [ "$?" -ne 0 ]; then
        echo "FAIL: $1 failed!!!"
    else
        echo "PASS: $1 passed."
    fi
}

# ------------------------   main  ----------------------------------

echo "Setup: Remove all containers and images."
podman rm -af
podman rmi -af

echo "Step1: pull image alpine and check"
pull_result=`${CMD_PREFIX}PullImage '{"name": "alpine"}'`
verify "Pull image with PullImage()"
echo "${pull_result}"
image_id=`echo ${pull_result} | grep -E '[[:alnum:]]{64}'`
verify "Image ID print"

echo
echo "Step2-1: check image with ListImages()"
list_result=`${CMD_PREFIX}ListImages`
verify "List image with ListImages()"
echo ${list_result} | grep "${image_id}"
verify "Image id print"

echo
echo "Step2-2: check image with GetImage()"
get_img_result=`${CMD_PREFIX}GetImage '{"name": "docker.io/library/alpine:latest"}'`
verify "Get image with GetImage()"
echo ${get_img_result} | grep "${image_id}"
verify "Image ID print"

echo
echo "Step3-1: create container with image alpine and check"
create_result=`${CMD_PREFIX}CreateContainer '{"create": {"image": "alpine", "name": "test", "command": ["/usr/bin/top"], "detach": true}}'`
verify "Create container test with CreateContainer()"
echo "${create_result}"
container_id=`echo ${create_result} | grep -E '[[:alnum:]]{64}'`
verify "Container ID print"

echo
echo "Step3-2: start container test"
start_result=`${CMD_PREFIX}StartContainer '{"name": "test"}'`
verify "Start container test with StartContainer()"
echo ${start_result} | grep "${container_id}"
verify "Container ID print"

echo
echo "Step-4: pause container test"
pause_result=`${CMD_PREFIX}PauseContainer '{"name": "test"}'`
verify "Pause container test with PauseContainer()"
echo ${pause_result} | grep "${container_id}"
verify "Container id print"

echo
echo "Step-5: unpause container test"
unpause_result=`${CMD_PREFIX}UnpauseContainer '{"name": "test"}'`
verify "Unpause container test with UnpauseContainer()"
echo ${unpause_result} | grep "${container_id}"
verify "Container ID print"

echo
echo "Step-6: stop container test with timeout 5s"
stop_result=`${CMD_PREFIX}StopContainer '{"name": "test", "timeout": 5}'`
verify "Stop container test with StopContainer()"
echo ${stop_result} | grep "${container_id}"
verify "Container ID print"

echo
echo "Step-7: remove container test"
rmv_result=`${CMD_PREFIX}RemoveContainer '{"name": "test"}'`
verify "Remove container test with RemoveContainer()"
echo ${rmv_result} | grep "${container_id}"
verify "Container ID print"

echo
echo "Step-8: Get a non-exist image"
geti_result=`${CMD_PREFIX}GetImage '{"name": "non-exist"}' 2>&1`
verify "Get a non-exist image"
echo ${geti_result} | grep "ImageNotFound"
verify "Error io.podman.ImageNotFound returned"

echo
echo "Step-9: Get a non-exist container"
getc_result=`${CMD_PREFIX}GetContainer '{"name": "non-exist"}' 2>&1`
verify "Get a non-exist container"
echo ${getc_result} | grep "ContainerNotFound"
verify "Error io.podman.ContainerNotFound returned"
```

```python tab="python-varlink"
#!/usr/bin/python3

"""
Test script with unitttest to verify pthon3-varlink package work well
with podman.

Prerequisites:
    1. Host must be RHEL8.
    2. Yum module container-tools is installed.

Test steps:
    1. Pull image. -->> varlink method: PullImage()
    2. Check image is pulled. -->> varlink method: ListImages(), GetImage()
    3. Run image. -->> varlink method: CreateContainer(), GetContainer()
    4. Pause container. -->> varlink method: PauseContainer()
    5. Unpause container. -->> varlink method: UnpauseContainer()
    6. Stop container. -->> varlink method: StopContainer()
    7. Start container. -->> varlink method: StartContainer()
    8. Remove container. -->> valink method: RemoveContainer()
    9. Remove image. -->> varlink method: RemoveImage(), GetImage()
    10. Get a non-exist image. -->> varlink error: ImageNotFound
    11. Get a non-exist container. -->> varlink error: ContainerNotFound
"""

import argparse
import unittest
import subprocess
import time

import varlink


class TestPyVarlink(unittest.TestCase):

    @classmethod
    def setUpClass(cls):
        addr = "unix:/run/podman/io.podman"
        interface = "io.podman"

        # Delete all containers
        subprocess.call('podman rm -af', shell=True)
        # Delete all images
        subprocess.call('podman rmi -af', shell=True)

        try:
            cls._client = varlink.Client.new_with_address(addr)
            cls.conn = cls._client.open(interface, namespaced=True)
        except varlink.ConnectionError as e:
            print("ConnectionError:", e)
            raise e
        except varlink.VarlinkError as e:
            print(e)
            print(e.error())
            print(e.parameters())
            raise e

    def _check_ctn_status(self):
        # Check container running states
        return self.conn.GetContainer('test').container.status

    def test_a_pull(self):
        self.assertEqual(len(self.conn.PullImage('alpine:latest').id), 64)
        time.sleep(3)

    def test_b_images_list(self):
        self.assertEqual(self.conn.ListImages().images[0].repoTags,
                         ['docker.io/library/alpine:latest'])

    def test_c_get_image(self):
        self.assertIsNotNone(self.conn.GetImage('docker.io/library/alpine:latest'))
        self.assertRaises(varlink.VarlinkError, self.conn.GetImage, 'somethingNotExist')

    def test_d_run_image(self):
        # Due to bugzilla 1648300, this test step is implemented by shell command
        exit_code = subprocess.call('podman run -d --name test alpine /usr/bin/top', shell=True)
        self.assertEqual(exit_code, 0)
        time.sleep(3)

        self.assertEqual(self._check_ctn_status(), 'running')

    def test_e_list_containers(self):
        self.assertEqual(self.conn.ListContainers().containers[0].names,
                         'test')

    def test_f_get_container(self):
        self.assertIsNotNone(self.conn.GetContainer('test'))
        self.assertRaises(varlink.VarlinkError, self.conn.GetContainer, 'somethingNotExist')

    def test_g_pause_container(self):
        self.assertEqual(len(self.conn.PauseContainer('test').container), 64)
        time.sleep(3)
        self.assertEqual(self._check_ctn_status(), 'paused')

    def test_h_unpause_container(self):
        self.assertEqual(len(self.conn.UnpauseContainer('test').container), 64)
        time.sleep(3)
        self.assertEqual(self._check_ctn_status(), 'running')

    def test_i_stop_container(self):
        self.assertEqual(len(self.conn.StopContainer('test').container), 64)
        time.sleep(10)
        self.assertEqual(self._check_ctn_status(), 'exited')

    def test_j_start_container(self):
        self.assertEqual(len(self.conn.StartContainer('test').container), 64)
        time.sleep(3)
        self.assertEqual(self._check_ctn_status(), 'running')

    def test_k_remove_container_force(self):
        self.assertEqual(len(self.conn.RemoveContainer('test', True).container), 64)
        time.sleep(3)
        self.assertEqual(len(self.conn.ListContainers().containers), 0)

    def test_l_remove_image(self):
        self.assertEqual(len(self.conn.RemoveImage('alpine').image), 64)
        self.assertEqual(len(self.conn.ListImages().images), 0)

    def test_m_get_non_exist_image(self):
        with self.assertRaises(VarlinkError) as context:
            self.conn.GetImage('non-exist')
        self.assertTrue('io.podman.ImageNotFound' in str(context.exception))

    def test_n_get_non_exist_container(self):
        with self.assertRaises(VarlinkError) as context:
            self.conn.GetContainer('non-exist')
        self.assertTrue('io.podman.ContainerNotFound' in str(context.exception))

    @classmethod
    def tearDownClass(cls):
        # Delete all containers
        subprocess.call('podman rm -af', shell=True)
        # Delete all images
        subprocess.call('podman rmi -af', shell=True)

        cls.conn.close()
        cls._client.cleanup()


if __name__ == '__main__':
    unittest.main(verbosity=2)
```
