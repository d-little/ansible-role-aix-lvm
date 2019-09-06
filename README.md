# ansible-role-aix-lvm

**Notice**: This role is undergoing heavy construction.  Not recommended for production use just yet.

An [Ansible](https://github.com/ansible/ansible) role to manage Volume Groups/Logical Volumes/Filesystems on AIX Servers.  It is currently designed to primarily build out and create new environments, while it is possible extend individual filesystems, it's not best suited for that job.  It excels at taking a bunch of PVs and defining all of the VGs, LVs, and FSs on the target host. This role was based off the excellent [ansible-manage-lvm](https://github.com/mrlesmithjr/ansible-manage-lvm) role from [@mrlesmithjr](https://github.com/mrlesmithjr/)

**NOTE**:

> * Can be used to create and destroy LVM Groups and volumes.
>   * I would not currently trust it to handle resizing/modifying... it's not fully tested.
> * Designed to define a 'whole' environment;  it's not really designed to create ad-hoc LV/FS etc.
>   * The primary use-case is to define the whole AIX server storage on an initial build.
> * Does not (*currently?*) handle Remote/NFS mount points.
> * If you have an 'overmounted' filesystem it will fail as the aix_filesystem module does not create the directory on mount
>   * ie: /foo/ and /foo/bar/ are both filesystems and created before being mounted
>   * /foo/ is mounted, and the /foo/bar/ directory is 'gone'
>   * When attempting to mount /foo/bar/ it will fail as the directory does not exist.

## Requirements

Devices/disks to be members of the LVM setup **must be** identified prior to using this role.

**NOTE**:

> * Ensure that you select the correct devices/disks.
> * To create a VG w/out creating LV's define lvname w/ value as `None`, as per the below example.

## Role Variable Structure

We define a list of dicts describing the VG's required

| Variables                      | Default                  | Comments |
| :---                           | :---                     | :--- |
| `manage_lvm`                   | `false`                  | Just a 'safety' mark to ensure this role should be run.  If `false`, LVM will not be managed by this role |
| `vglist` (required)            | -                        | A list of `vglist`, objects, defined below |

### `vglist`

A complete overview of `vglist` options follows below.  Defaults are per [aix_lvg module documentation](https://docs.ansible.com/ansible/latest/modules/aix_lvg_module.html).  **NB**: Default Values in this role will *always* use the module defaults; if the `aix_lvg` module changes default values this document *may not be up-to-date in which default values are being used*.  Please log an issue or PR if the `aix_lvg` values change and this document has not been updated.

| Variables                      | Default                  | Comments |
| :---                           | :---                     | :--- |
| `vgname` (required)            | -                        | Name of the VG |
| `disks` (required)             | -                        | List of names of PV devices |
| `force`                        | `false`                  | Forcefully create VG |
| `ppsize`                       | -                        | Size of the VG Physical Partitions |
| `state`                        | `present`                | One of `absent` or `present`.  If `absent`, all underlying FSs and LVs must be removed from the system  first. |
| `vg_type`                      | `normal`                 | Type of VG; one of `big`, `normal`, `scalable`  |
| `lvlist`                       | -                        | `lvlist` object, defined below.  Any number of these can be set. |

### `lvlist`

A complete overview of `lvlist` options follows.  Any number of these can be defined within the `vglist` object above.  Defaults are [aix_lvol module documentation](https://docs.ansible.com/ansible/latest/modules/aix_lvol_module.html) and [aix_filesystem](https://docs.ansible.com/ansible/latest/modules/aix_filesystem_module.html).  While it is possible to create a FS without an existing LV, this role takes a 'best-practice' approach that the parent LV is created prior to every FS. **NB**: Default Values in this role will *always* use the module defaults; if the `aix_lvol` or `aix_filesystem` modules change default values this document *may not be up-to-date in which default values are being used*.  Please log an issue or PR if the default values change and this document has not been updated.

| Variables                      | Default   | Comments |
| :---                           | :---      | :--- |
| `lvname` (required)            | -         | Name of the LV |
| `lvstate`                      | `present` | One of `present` or `absent`.  If `absent`, LV and any associated FS will be removed.
| `lvcopies`                     | `1`       | Number of copies for the LV.  Max is 3
| `lvtype`                       | `jfs2`    | What [type of LV](https://www.ibm.com/support/knowledgecenter/ssw_aix_72/com.ibm.aix.cmds3/mklv.htm#mklv__row-d3e142707) to create: [`jfs2`, `jfs`, `paging`, `etc`] |
| `lvopts`                       | -         | Free-form [options to be passed to mklv](https://www.ibm.com/support/knowledgecenter/ssw_aix_72/com.ibm.aix.cmds3/mklv.htm)
| `lvpolicy`                     | `maximum` | Sets interphysical volume allocation policy, one of [`maximum`, `minimum`].
| `lvpvs`                        | -         | A list of which PVs to use in the host VG. |
| `lvsize`                       | -         | Size of the logical volume with one of the [MGT] units. |
| `lvstate`                      | `present` |  One of `asbsent` or `present`. If `present`, `lvsize` is required. |
| `fsaccount_subsystem`          | `false`   | Is the FS to be processed by the accounting subsystem. Bool. |
| `fsattributes`                 | `"agblksize='4096',isnapshot='no'"` | Attributes for files system separated by comma |
| `fsauto_mount`                 | `false`   | File system is automatically mounted at system restart. |
| `fsfilesystem`                 | -         | The Mount Point, directory where the filesystem will be mounted. `required` if `fsstate` is not `absent` |
| `fsstate`                      | `present`   | One of [`present`, `absent`, `mounted`, `unmounted`].  If you want to create an LV with no attached FS, set `fsstate` to `absent` |
| `fsmount_group`                | -         | FS mount group
| `fspermissions`                | `rw`      | FS permissions; One of [`rw`, `ro`]  |

## Dependencies

None

## Example Playbooks

### Small, simple experimental VG

```yaml
---
- hosts: test-nodes
  vars:
    manage_lvm: true
    vglist:
      - vgname: vg_exp1
        disks:
          - hdisk4
        force: true
        lvlist:
          - lvname: lv_exp1
            lvcopies: 1
            lvsize: 5G
            fsauto_mount: true
            fsstate: mounted
            fsfilesystem: /mnt/exp1
          - lvname: lv_exp2
            lvcopies: 1
            lvsize: 10G
            fsauto_mount: true
            fsstate: mounted
            fsfilesystem: /mnt/exp2
  roles:
    - role: d-little.aixlvm
  tasks:
```

### One VG, Multiple LV

Note: This is a normal use case of this role

```yaml
---
- hosts: test-nodes
  vars:
    manage_lvm: true
    vglist:
      - vgname: vg_test1
        disks:
          - hdisk10
          - hdisk11
        lvlist:
          - lvname: lv_test1_1
            lvcopies: 2
            lvsize: 5G
            fsauto_mount: true
            fsstate: mounted
            fsfilesystem: /mnt/test1_1
          - lvname: lv_test1_2
            lvcopies: 2
            lvsize: 10G
            fsauto_mount: true
            fsstate: mounted
            fsfilesystem: /mnt/test1_2
  roles:
    - role: d-little.aixlvm
  tasks:
```

### Empty VGs with no LVs

```yaml
---
- hosts: test-nodes
  vars:
    manage_lvm: true
    vglist:
      - vgname: vg_test2_1
        disks:
          - hdisk20
          - hdisk21
        lvlist:
          None
      - vgname: vg_test2_2
        disks:
          - hdisk22
          - hdisk23
        lvlist:
          None
  roles:
    - role: d-little.aixlvm
  tasks:
```

### One VG, One LV, All Variables Defined

This is not a normal use case, but just in case you want to change everything

```yaml
---
- hosts: test-nodes
  vars:
    manage_lvm: true
    vglist:
      - vgname: vg_test3
        force: true
        disks:
          - hdisk30
          - hdisk31
        ppsize: 128
        state: present
        vg_type: big
        lvlist:
          - lvname: lv_test3
            lvstate: present
            lvcopies: 2
            lvopts: aaaa
            lvpolicy: maximum
            lvpvs: hdisk30, hdisk31
            lvsize: 50G
            lvstate: present
            fsaccount_subsystem: false
            fsattributes: aaaaa
            fsauto_mount: true
            fsfilesystem: /mnt/test3_1
            fsstate: mounted
            fsmount_group: aaaa
            fspermissions: rw
  roles:
    - role: ansible-manage-lvm
  tasks:
```

## Authors

* *David Little* - *Initial work* - [d-little](https://github.com/d-little/)

## License

MIT

## Acknowledgments

* Larry Smith Jr. - [mrlesmithjr](https://github.com/mrlesmithjr)
* [Ansible](https://www.ansible.com)
