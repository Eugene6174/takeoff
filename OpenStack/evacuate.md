### API

### Overview

### Workflow

**This is bases on Newton**

**1 nova-api(nova.compute.api.API.evacuate)**
* check vm_state in [active, stopped, error]
* check source host nova-compute service is down
* save instance task_state to **rebuilding** and except task_state is None before
* create migration object with status of **accepted**
* notify about instance usage "compute.instance.evacuate"
* rpc cast rebuild instance to nova-conductor **goto 2**

**2 nova-conductor**
* rpc call nova-scheduler select destination **goto 3** to select a host if not host passed from rest api
* notify about instance usage "compute.instance.rebuild.scheduled"
* rpc cast destination nova-compute to rebuild instance **goto 4**

**3 nova-scheduler**
* select destination

**4 nova-compute destination(nova.compute.manager.ComputeManager.rebuild_instance)**
* get rebuild claim of resource tracer
* rebuild resource claim and save migration status to **pre-migrating**
* check compute driver support recreate
* call compute driver to check instance exist on host
* check instance on share storage
* notify about instance usage "compute.instance.rebuild.start"
* save instance power_state to current power state and task_state to **rebuilding** except task_state is rebuilding before
* call network api to setup network info on destination host
* call network api to get instance network info
* do rebuild default implement
  * detach volumes
    * call cinder api to terminate volume connection
    * disconnect volume
  * save instance task_state to **rebuilding_block_device_mapping**
  * attach volumes
    * call cinder api to initialize connection
    * connect volume
  * save instance task_state to **rebuild_spawning**
  * call compute driver to do spawn

* save instance **task_state to None** and vm_state to **active**
* stop instance if previous vm state is stopped
* update scheduler instance info
* notify about instance usage "compute.instance.rebuild.end"
* **save instance host to destination host**
* save migration status to **done**

**5 nova-compute source(nova.compute.manager.ComputeManager.init_host)**
* destroy evacuated instances
  * get evacuate list by filter status in **[accepted, done]** and source host is this compute host
  * call compute driver to list instances on hypervisor
    * call libvirt to list all defined domains
  * check instance storage shared
  * call compute driver to destory evacuated instances
    * destory domain
    * unplug vifs
    * unfilter instance
    * disconnect volume
    * undefine domain
  * save migration status to **completed**
