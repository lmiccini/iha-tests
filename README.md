Playbooks meant to run via test-operator.


Example AnsibleTest cr:

~~~
---
apiVersion: test.openstack.org/v1beta1
kind: AnsibleTest
metadata:
  name: iha-tests
  namespace: openstack
spec:
  containerImage: quay.io/podified-master-centos9/openstack-ansible-tests:current-podified
  debug: false
  storageClass: "lvms-local-storage"
  workloadSSHKeySecretName: 'test-operator-controller-priv-key'
  ansiblePlaybookPath: playbooks/main.yaml
  ansibleGitRepo: https://github.com/openstack-k8s-operators/iha-tests
  ansibleGitBranch: main
  ansibleInventory: |
    localhost ansible_connection=local ansible_python_interpreter=python3
    controller-0 ansible_host=192.168.111.9 ansible_user=zuul ansible_ssh_private_key_file=~/test_keypair.key ansible_host_key_checking=false
    hypervisor ansible_host=192.168.111.1 ansible_user=zuul ansible_ssh_private_key_file=~/test_keypair.key ansible_host_key_checking=false
  ansibleVarFiles: |
    ---
    # evacuation delay
    delay: 0
~~~

## Results tracking

Each test should create a results file under /home/zuul/iha-tests-results/ named after the respective playbook, for example "01_disabled.xml".

~~~
{% if success %}
    <testcase classname="{{ name }}" name="{{ name }}">
    </testcase>
{% else %}
    <testcase classname="{{ name }}" name="{{ name }}">
        <failure type="failure"> tests failed </failure>
    </testcase>
{% endif %}
~~~

Right now we use the following as first task to pre-set the test to have failed:

~~~
---
- name: 01 TEST DISABLED PARAM
  hosts: controller-0
  vars:
    name: "01_disabled"
  tasks:
    - name: 01 Create results file (failure)
      ansible.builtin.template:
        src: templates/iha-tests-results.xml.j2
        dest: /home/zuul/iha-tests-results/{{ name }}.xml
      vars:
        success: false
~~~

And last task instead sets the test to have succeeded if everything went fine:

~~~
    - name: 01 Create results file (success)
      ansible.builtin.template:
        src: templates/iha-tests-results.xml.j2
        dest: /home/zuul/iha-tests-results/{{ name }}.xml
      vars:
        success: true
~~~

The 99_gen_junitxml.yaml playbook will generate a junit.xml file out of them.

# TESTS

### note

Tests typically fetch compute hostnames and save them as facts for use throughout the test. This allows targeting specific compute nodes by variable reference.


## 00_DEPLOY

goal: deploy instanceha operator resources.

1. discover all compute VMs on the hypervisor via `virsh list`
2. fetch UUIDs for each compute VM
3. build fencing entries dynamically (compute-1 uses ipmi, others use redfish)
4. template and apply fencing-secret, iha-cm (configmap), and iha CR
5. wait for instanceha pod to be ready
6. install prerequisite ansible collections (openstack.cloud, community.libvirt)


## 00_PREP

goal: prepare environment for instanceha tests.

1. install openstackclient and osc-placement packages
2. create clouds.yaml configuration file
3. create openstack network resources (ihanet, subnets, routers, security groups)
4. download and create cirros test image
5. create iha.nano flavor


## 01_DISABLED

goal: verify DISABLED=true prevents evacuation while logging compute failures.

1. set DISABLED=true
2. verify parameter is set
3. create vm c1 on compute-0
4. crash compute-0 and verify c1 is not evacuated
5. verify instanceha logs contain warning about disabled evacuation
6. set DISABLED=false
7. verify c1 is evacuated after instanceha restarts
8. cleanup


## 02_TAG_FLAVOR

goal: verify TAGGED_FLAVORS=true evacuates only vms created from flavors with EVACUABLE_TAG.

1. create flavor iha.nanoha and tag with trait:CUSTOM_HA=true
2. create CUSTOM_HA trait and set on resource providers
3. create vm (non-tagged flavor) and vm-ha (tagged flavor) on compute-0
4. crash compute-0
5. verify vm was not evacuated
6. verify vm-ha was evacuated
7. cleanup


## 03_TAG_IMAGE

goal: verify TAGGED_IMAGES=true evacuates only vms created from images with EVACUABLE_TAG.

1. create image cirros-ha and tag with trait:CUSTOM_HA
2. create CUSTOM_HA trait and set on resource providers
3. create vm (non-tagged image) and vm-ha (tagged image) on compute-0
4. crash compute-0
5. verify vm was not evacuated
6. verify vm-ha was evacuated
7. cleanup


## 04_TAG_AGGREGATE

goal: verify TAGGED_AGGREGATES=true evacuates only vms on aggregates with EVACUABLE_TAG.

1. create aggregate iha (compute-0) and nonha (compute-1, compute-2)
2. tag iha aggregate with trait:CUSTOM_HA=true
3. create vm-ha on compute-0 and vm on compute-1
4. crash compute-0 and verify vm-ha was evacuated
5. crash compute-1 and verify vm was not evacuated
6. cleanup


## 05_LEAVE_DISABLED

goal: verify LEAVE_DISABLED=true prevents compute restart after evacuation.

1. set LEAVE_DISABLED=true
2. create vm c1 on compute-0
3. crash compute-0
4. verify c1 was evacuated
5. verify compute-0 remains down
6. manually recover compute-0
7. cleanup


## 06_RESERVED

goal: verify RESERVED_HOSTS=true enables a reserved compute before evacuation.

1. set RESERVED_HOSTS=true
2. disable compute-1 with reason "reserved"
3. create vm c1 on compute-0
4. crash compute-0
5. verify c1 was evacuated and compute-1 was enabled
6. cleanup


## 07_SMARTEVAC

goal: verify SMART_EVACUATION=true monitors evacuations and logs failures after multiple attempts.

1. set SMART_EVACUATION=true
2. create vm c1 on compute-0
3. crash compute-0
4. verify c1 was evacuated and logs show "evacuated successfully"
5. delete c1
6. create large flavor (1 vm per compute) to simulate capacity constraints
7. create vmfill1 on compute-1, vmfill2 on compute-2, vm0 on compute-0
8. crash compute-0
9. verify vm0 cannot evacuate due to capacity
10. verify logs contain "Failed evacuating" message
11. cleanup


## 08_SELECTIVE_TAGS

goal: verify TAGGED_FLAVORS and TAGGED_IMAGES can be independently disabled.

1. create tagged flavor iha.nanoha and tagged image cirros-ha
2. set CUSTOM_HA trait on resource providers
3. create vm-flavorha and vm-imageha on compute-0
4. crash compute-0 and verify both vms evacuated
5. set TAGGED_FLAVORS=false
6. create vm-flavorha and vm-imageha on compute-0
7. crash compute-0 and verify only vm-imageha evacuated (flavor tagging disabled)
8. set TAGGED_IMAGES=false
9. create vm-flavorha and vm-imageha on compute-0
10. crash compute-0 and verify only vm-flavorha evacuated (image tagging disabled)
11. cleanup


## 09_KDUMP

goal: verify CHECK_KDUMP=true waits for kdump completion before evacuation.

1. set CHECK_KDUMP=true
2. attach instanceha pod to internalapi network
3. reconfigure compute network interfaces for kdump compatibility (vlan instead of ovs)
4. configure kdump on compute-0 with fence_kdump pointing to instanceha pod
5. create vm c1 on compute-0
6. crash compute-0
7. verify evacuation occurs after kdump completes
8. verify logs show compute recognized as kdumping
9. cleanup


## 10_STRESSVMS

goal: verify instanceha handles evacuation of multiple vms simultaneously.

1. ensure all compute nodes are running
2. set quota to allow 100 instances
3. delete any existing instances
4. create configurable number of vms (default 1) on each compute node
5. crash compute-0
6. verify all vms from compute-0 evacuated to other computes and reached ACTIVE state
7. cleanup all vms and wait for compute-0 recovery


## 12_MAINTENANCE

goal: verify instanceha ignores compute nodes in maintenance mode.

1. create vm c1 on compute-0
2. disable compute-0 with reason "maintenance"
3. crash compute-0
4. verify c1 was not evacuated
5. verify compute-0 remains disabled
6. restart compute-0
7. verify compute-0 returns to up state
8. re-enable compute-0
9. cleanup


## 13_SKIP_SERVER_NAME

goal: verify SKIP_SERVERS_WITH_NAME excludes matching vms from evacuation.

1. set SKIP_SERVERS_WITH_NAME to a pattern matching specific vms
2. create matching and non-matching vms on compute-0
3. crash compute-0
4. verify matching vms were not evacuated
5. verify non-matching vms were evacuated
6. cleanup


## 14_THRESHOLD

goal: verify THRESHOLD prevents mass evacuation when too many computes are down.

1. set THRESHOLD to a percentage value
2. create vms on multiple computes
3. crash enough computes to exceed the threshold
4. verify evacuation is blocked and logs show threshold exceeded
5. cleanup


## 15_ORCHESTRATED

goal: verify ORCHESTRATED_RESTART=true performs priority-based evacuation.

1. set ORCHESTRATED_RESTART=true
2. create vms on compute-0
3. crash compute-0
4. verify evacuation follows priority ordering
5. verify logs show orchestrated restart behavior
6. cleanup


## 16_SHUTOFF_ERROR

goal: verify instanceha preserves vm state during evacuation (SHUTOFF and ERROR vms).

1. create vms on compute-0 with different states (ACTIVE, SHUTOFF, ERROR)
2. crash compute-0
3. verify ACTIVE vms are evacuated and become ACTIVE
4. verify SHUTOFF vms are evacuated and remain SHUTOFF
5. verify ERROR vms are handled correctly
6. cleanup


## 17_RESERVED_ADVANCED

goal: verify advanced reserved host features including FORCE_RESERVED_HOST_EVACUATION, aggregate matching, and zone matching.

### TEST 1: Aggregate Matching with FORCE_RESERVED_HOST_EVACUATION

1. create aggregate 'prod' containing compute-0 and compute-1
2. create aggregate 'dev' containing compute-2
3. tag both aggregates with trait:CUSTOM_HA=true
4. set RESERVED_HOSTS=true, TAGGED_AGGREGATES=true, FORCE_RESERVED_HOST_EVACUATION=true
5. set compute-1 as reserved (same aggregate as compute-0)
6. create vm1 on compute-0
7. crash compute-0
8. verify vm1 is evacuated specifically to compute-1 (not compute-2)
9. verify compute-1 was enabled from reserved pool

### TEST 2: Zone Matching with FORCE_RESERVED_HOST_EVACUATION

1. remove previous aggregates
2. create availability zone 'zone-a' containing compute-0 and compute-1
3. create availability zone 'zone-b' containing compute-2
4. set RESERVED_HOSTS=true, TAGGED_AGGREGATES=false, FORCE_RESERVED_HOST_EVACUATION=true
5. restart and recover compute-0
6. set compute-1 as reserved (same zone as compute-0)
7. create vm2 on compute-0 in zone-a
8. crash compute-0
9. verify vm2 is evacuated specifically to compute-1 (same zone)
10. verify compute-1 was enabled from reserved pool

### TEST 3: No Matching Reserved Host Scenario

1. enable compute-1 and set compute-2 as reserved (different zone - zone-b)
2. restart and recover compute-0
3. create vm3 on compute-0 in zone-a
4. crash compute-0
5. verify vm3 is evacuated (scheduler chooses destination, no forced target)
6. verify instanceha logs show "no reserved compute found" warning
7. verify compute-2 was NOT enabled (wrong zone)
8. cleanup aggregates and vms

### TEST 4: Partial Evacuation Failure with Reserved Host

1. create capacity-filling vms so reserved host cannot fit all evacuating vms
2. crash compute-0 with multiple vms
3. verify some vms evacuate to the reserved host
4. verify remaining vms fail gracefully
5. cleanup


## 18_K8S_EVENTS

goal: verify instanceha emits Kubernetes events for key actions.

1. create vm on compute-0
2. crash compute-0
3. verify K8s events are emitted (e.g. EvacuationStarted, EvacuationCompleted)
4. check event fields contain correct metadata
5. cleanup


## 19_HEALTHCHECK

goal: verify instanceha health and readiness probes respond correctly.

1. check readiness probe endpoint returns healthy
2. check health probe endpoint returns healthy
3. verify probes reflect actual instanceha state
4. cleanup


## 20_EVACUATION_RETRY

goal: verify evacuation retry loop with SMART_EVACUATION.

1. set SMART_EVACUATION=true with EVACUATION_RETRIES
2. create vm on compute-0
3. crash compute-0
4. verify evacuation is retried on failure
5. verify logs show retry attempts
6. cleanup


## 21_CONTINUE_ON_FAILURE

goal: verify partial evacuation continues when some vms fail to evacuate.

1. create multiple vms on compute-0 with different flavors (some too large for remaining capacity)
2. crash compute-0
3. verify vms that can be evacuated are evacuated
4. verify vms that cannot be evacuated are reported as failed
5. verify evacuation continues past individual failures
6. cleanup


## 22_RECONCILIATION

goal: verify orphaned fenced host recovery (reconciliation loop).

1. fence a compute host externally (outside instanceha)
2. verify instanceha detects the orphaned fenced host
3. verify instanceha recovers the host automatically
4. cleanup


## 23_HEARTBEAT

goal: verify CHECK_HEARTBEAT-based host filtering.

1. set CHECK_HEARTBEAT=true with HEARTBEAT_TIMEOUT
2. Phase 1: test with pod IP connectivity
3. Phase 2: test with LoadBalancer IP connectivity
4. verify heartbeat-based filtering works correctly
5. cleanup


## 24_APPLICATION_CREDENTIALS

goal: verify instanceha works with application credential authentication.

1. create application credential in OpenStack
2. create application credential secret in K8s
3. configure instanceha to use applicationCredentialSecret
4. create vm on compute-0
5. crash compute-0
6. verify evacuation works with application credential auth
7. cleanup


## 25_PROMETHEUS_METRICS

goal: verify instanceha exposes Prometheus metrics via telemetry-operator.

1. collect baseline metrics from instanceha
2. create vm on compute-0
3. crash compute-0
4. verify post-evacuation metrics are updated (evacuation counters, timing, etc.)
5. cleanup


## 26_THRESHOLD_TAGGED_AGGREGATES

goal: verify threshold behavior scoped to tagged aggregates.

### TEST 1: Below Threshold (should evacuate)

1. create aggregate 'evacuable-agg' containing compute-0 through compute-3
2. tag aggregate with trait:CUSTOM_HA=true
3. set TAGGED_AGGREGATES=true, THRESHOLD=50
4. create vm on compute-0
5. crash compute-0 (1 of 4 = 25%, below 50% threshold)
6. verify vm is evacuated

### TEST 2: Above Threshold (should block)

1. recover compute-0
2. create vms on compute-0, compute-1, compute-2
3. crash compute-0, compute-1, compute-2 (3 of 4 = 75%, above 50% threshold)
4. verify evacuation is blocked
5. verify logs show threshold exceeded
6. verify ThresholdExceeded K8s event is emitted
7. cleanup
