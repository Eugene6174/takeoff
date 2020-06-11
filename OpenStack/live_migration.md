### [API](https://docs.openstack.org/api-ref/compute/?expanded=live-migrate-server-os-migratelive-action-detail#live-migrate-server-os-migratelive-action)

>/servers/{server_id}/action

Live-migrates a server to a new host without rebooting.

Specify the os-migrateLive action in the request body.

Use the host parameter to specify the destination host. If this param is null, the scheduler chooses a host. If a scheduled host is not suitable to do migration, the scheduler tries up to migrate_max_retries rescheduling attempts.

Starting from API version 2.25, the block_migration parameter could be to auto so that nova can decide value of block_migration during live migration.

Policy defaults enable only users with the administrative role to perform this operation. Cloud providers can change these permissions through the policy.json file.

Starting from REST API version 2.34 pre-live-migration checks are done asynchronously, results of these checks are available in instance-actions. Nova responds immediately, and no pre-live-migration checks are returned. The instance will not immediately change state to ERROR, if a failure of the live-migration checks occurs.

Starting from API version 2.68, the force parameter is no longer accepted as this could not be meaningfully supported by servers with complex resource allocations.

Normal response codes: 202

Error response codes: badRequest(400), unauthorized(401), forbidden(403) itemNotFound(404), conflict(409)

**Example Live-Migrate Server (os-migrateLive Action)**
```
{
    "os-migrateLive": {
        "host": "01c0cadef72d47e28a672a76060d492c",
        "block_migration": "auto",
        "force": false
    }
}
```

### Overview


### Workflow

**1 nova-api(nova.compute.api.API.live_migrate)**
* check vm_state in [active, puased]
* save instance task_state to **migrating**
* get request_spec
* check force and destination host name
* rpc cast **live_migrate_instance(async)** to nova-conductor if api version > 2.34 else rpc call **migrate_server(sync)**  **goto 2**

**2 nova-conductor**
* check instance is active
* check source host is up
* without destination
  * rpc call nova-scheduler select destination hosts **goto 3**
  * check destination host compatible with source hypervisor
  * rpc call destination nova-compute to check can live migration destination **goto 4**
* with destination
  * check destination is not source
  * check destination nova-compute host is up
  * check destination has enough memory
  * check destination host compatible with source hypervisor
  * rpc call destination nova-compute to check can live migration destination **goto 4**
* rpc cast source nova-compute to live migrate instance **goto 6**

**3 nova-scheduler**
* select destination hosts

**4 nova-compute destination(nova.compute.mananger.ComputeManager.check_can_live_migration_destination)**
* driver check can live migrate at destination
  * compare cpu
  * create share storage test file
* rpc call source nova-compute to check can live migration at source **goto 5**
* driver cleanup live migration destination check
  * cleanup share storage test file

**5 nova-compute source(nova.compute.mananger.ComputeManager.check_can_live_migrate_source)**
* check is volume backend instance
* driver check can live migrate at source
  * check share storage test file
  * check is share storage block
  * check destination host has enough disk if block migrate

---

**6 nova-compute source(nova.compute.mananger.ComputeManager.live_migration)**
* save migration status to "queued"
* save migration status to "preparing"
* call destination nova-compute to do pre_live_migration **goto 7**
* save migration status to "running"
* driver live migration
  * get instance guest domain
  * copy live migration disks if block migration
  * set libvirt max live migration speed to 1
  * spawn a green thread to do live migration operation
    * call libvirt migrate to start live migration
  * wait neutron vif_plugged_complete event
  * live migrate monitor
    * start a loop to get libvirt live migrate job info and check if live migration completed
    * call post method if job completed -> **post live migration**
    * call recover method if job cancelled or failed -> **rollback live migration**
* post live migration
  * driver post live migration
    * call cinder api to initialize volume connection
    * disconnect volumes
  * call cinder api to terminate volume connection
  * get instance network info
  * notify about instnace usage "compute.instance.live_migration._post.start"
  * driver unfilter instance
  * call network api migrate instance start
  * driver post migration at source
    * unplug vifs
  * rpc cast post live migration at destination **goto 8**
  * get live migration cleanup flags
  * driver cleanup
    * unfilter instances
    * call libvirt destory instance if its power is not shutdown
    * disconnect volumes
    * call libvirt undefine domain
  * clear events for instance
  * update available resource
  * update scheduler instance info
  * clean instance console token
  * save migration status to "completed"
* rollback live migration
  * save instance **task_state to None**
  * call network api to setup network on source host
  * rpc call destination nova-compute remove volume connection **goto 9**
  * notify about instance usage "compute.instance._rollback.start"
  * get live migration cleanup flags
  * rpc cast rollback live migration at destination **goto 10**
  * notify about instance usage "compute.instance._rollback.end"
  * save migration status to "error" or "cancelled"

**7 nova-compute source(nova.compute.mananger.ComputeManager.pre_live_migration)**
* call cinder api to initialize volume connection
* call network api to get instance network info
* notify about instance usage "compute.instance.live_migraton.pre.start"
* driver pre live migration
  * remove and recreate instance path dir if not share instance path
  * create image and backing if not share block storage
  * connect volume
* call network api to setup network on destination host
* driver ensure filtering rules for instance
* notify about instance usage "compute.instance.live_migration.pre.end"

**8 nova-compute source(nova.compute.mananger.ComputeManager.post_live_migration_at_destination)**
* call network api to setup network on destination host
* call network api migrate instance finish on destination
* call network api get instance network info
* notify about instance usage "compute.instance.live_migration.post.dest.start"
* driver post live migration at destination
  * get libvirt guest domain
  * get instance libvirt domain xml
  * write domain xml(libvirt define domain)
* save instance **task_state to None, power_state to current power state and host to destination host**
* call network api to setup network on source host -> tear down
* notify about instance usage "compute.instance.live_migration.post.dest.end"

**9 nova-compute source(nova.compute.mananger.ComputeManager.remove_volume_connection)**
* driver detach volume
  * detach from libvirt domain
  * disconnect volume
* call cinder api to terminate volume connection

**10 nova-compute source(nova.compute.mananger.ComputeManager.rollback_live_migration_at_destination)**
* call network api to get instance network info
* notify about instance usage "compute.instance.live_migration.rollback.dest.start"
* call network api to setup network on destination host -> tear down
* driver rollback live migration on destination
  * call libvirt to destory instance domain
  * call libvirt undefine instance domain
  * unplug vifs
  * disconnect volume
* notify about instance usage "compute.instance.live_migration.rollback.dest.end"
