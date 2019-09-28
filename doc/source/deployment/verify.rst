.. _verifyinstallation:

Verify OpenStack Operation
==========================

The cloud deployment includes Rally testing for the core Airship UCP and
OpenStack services by default.

At this point, your `Deployer` node should have an OpenStack configuration file,
and the OpenStackClient (OSC) command line interface should be installed.

Test access to the OpenStack service via the VIP and determine that the OpenStack
services are functioning as expected by running the following commands:

.. code-block:: console

   export OS_CLOUD='openstack'
   openstack endpoint list
   openstack server list

OpenStack Tempest Testing
=========================

After the deployment of SUSE Containerized OpenStack has completed, it is possible to run
OpenStack Tempest tests against the core services in the deployment using the `run.sh` script.
As part of Tempest tests execution, there is need to configure OpenStack network resources
and provide a few configuration parameters in the `${WORKDIR}/env/extravars file`.

Setting Up Network CIDR(s) in OpenStack
----------------------------------------

Tempest will create a subnet pool (10.0.0.0/8) to use as the default network, and it will
need to know the CIDR block from which to allocate project IPv4 subnets. This value should be
specified with the following key in the `${WORKDIR}/env/extravars` file if customization is required
otherwise it will default to 10.0.0.0/8:

.. code-block:: yaml

   openstack_tempest_subnet_pool_cidr: "10.0.4.0/8"

Tempest will also create and external network which is needed by tempest tests execution.
This value should be specified with the following key in the `${WORKDIR}/env/extravars` file
if customization is required otherwise it will default to 172.24.4.0/24:

.. code-block:: yaml

   openstack_tempest_public_subnet_cidr: "192.168.100.0/24"

Tempest test execution via `run.sh` script will create the necessary networks as long as
above CIDR value is correctly identified and specified in extravars. The same networks will be
removed after tempest has run

.. note::

   As long as there is not an overlap of networks, the defaults should work out of the box
   given that the underlying networking is correctly configured.



Configuring Tempest Test Parameters
-----------------------------------

By default, the implementation of Tempest in SUSE Containerized OpenStack will run smoke tests
for all deployed services including compute, identity, image, network, and volume, using 4
workers.

To modify the number of workers, add the following key with a value of your choosing to the
extravars file:

.. code-block:: yaml

   tempest_workers: 6

To disable tests for specific OpenStack components, any or all of the following keys can be
added to the extravars file:

.. code-block:: yaml

   tempest_enable_cinder_service: false
   tempest_enable_glance_service: false
   tempest_enable_nova_service: false
   tempest_enable_neutron_service: false

To run all Tempest tests instead of just smoke tests, add the following key to the extravars
file:

.. code-block:: yaml

   tempest_test_type: "all"

Using a Blacklist
-----------------

To exclude specifc tests from the collection of tests being run against the deployment, they
can be added to the blacklist file located at

.. code-block:: console

   socok8s/playbooks/roles/airship-deploy-tempest/files/tempest_blacklist

When adding tests to the blacklist, each test should be listed on a new line and should be
formatted like the following example:

.. code-block:: console

   - (?:tempest\.api\.identity\.v3\.test_domains\.DefaultDomainTestJSON\.test_default_domain_exists)

By default, the blacklist file provided with SUSE Containerized OpenStack will be used when
running Tempest tests. If desired, use of a blacklist can be disabled by adding the following key
to ${WORKDIR}/env/extravars:

.. code-block:: yaml

   use_blacklist: false

Running Tempest Tests
---------------------

Once all of the OpenStack network resources have been created and all configuration parameters have
been provided in ${WORKDIR}/env/extravars, Tempest testing can be started by running the following
command from the root of the socok8s directory:

.. code-block:: console

   ./run.sh test

Once the Tempest pods have been deployed, testing will begin immediately. You can check the progress
of the test pod at any time by running

.. code-block:: console

   kubectl get pods -n openstack | grep tempest-run

Example output:

.. code-block:: console

   airship-tempest-run-tests-hq6jg                          1/1     Running       0          33m

A status of 'Running' indicates that testing is still in progress. Once testing is complete, the status
of the airship-tempest-run-tests pod will change to 'Complete', indicating that all *enabled* tests
are executed.

Tempest Test Results
--------------------

By default, tempest test execution pod logs are displayed on ansible stdout during `test` option.

Later, all test results can be viewed by retrieving the logs from the airship-tempest-run-tests pod by
running the following command:

.. code-block:: console

   kubectl logs -n openstack airship-tempest-run-tests-hq6jg

.. note::

   The logs can be viewed at any time, even while a current test batch is still running.

Once testing is complete, the logs will conclude with a summary of all passed, skipped, and failed tests
similar to the following:

.. code-block:: console

  Sample output for smoke tests execution (default value for tempest_test_type)

  ======
  Totals
  ======
  Ran: 120 tests in 1043.0000 sec.
   - Passed: 88
   - Skipped: 28
   - Expected Fail: 0
   - Unexpected Success: 0
   - Failed: 4
  Sum of execute time for each test: 1684.2065 sec.

  ==============
  Worker Balance
  ==============
   - Worker 0 (25 tests) => 0:06:17.321190
   - Worker 1 (39 tests) => 0:15:52.956097
   - Worker 2 (27 tests) => 0:17:23.015459
   - Worker 3 (29 tests) => 0:05:19.495695

