---
title: Container support
category: feature
authors: fromani
wiki_category: Feature
wiki_title: Container support
wiki_revision_count: 6
wiki_last_updated: 2015-12-15
feature_name: Container support
feature_modules: vdsm, engine?
feature_status: Planning
---

# Container support

### Summary

Add containers support to oVirt, to run containers on virtualization hosts, alongside VMs. Support to run containers-inside-VMs, all managed by oVirt is out of scope of this feature.

### Owner

*   Name: [ Francesco Romani](User:fromani)

<!-- -->

*   Email: <fromani@redhat.com>

### Detailed Description

The scope of this feature is not to provide a full-fledged container management system, but rather to add run containers alongside VMs.
The container should run seamlessly side to side with plain VMs, leveraging the existing management infrastructure of oVirt Engine.
The Containers will be represented as bare-bones VMs with minimal feature set (e.g. no migrations). The administrator will be able
to use different container runtimes (see below for details).
Future development includes the ability of running containers inside VMs, all managed by oVirt.
Containers and VMs must be created differently and cannot be converted from each other.

### Benefit to oVirt

The ability of running containers will give oVirt greater flexibility, making it possible to leverage transparently the best solution for any given circumstance. Sometimes VMs are the best tool for a job, sometimes containers are, sometimes one can need both at the same time. oVirt could be the most comprehensive solution in this regard. Please note that this feature will not shift the focus of oVirt, which will still be toward VM management.

### Dependencies / Related Features

The feature will be optionally enabled. If Vdsm reports the availability of supported runtime containers, the Engine will allow
the administrator to run container on a given host.
We will not add hard dependency on either Vdsm or oVirt Engine on container runtime support. Supported runtimes will be initially
[rkt](https://github.com/coreos/rkt) and later [runc](https://github.com/opencontainers/runc). See discussion below for details.

### Overview and design goals

This feature should be completely opt-in, should be completely transparent to the other flows and should require minimal changes to the current infrastructure.

#### The feature should be opt-in
Vdsm should detect automatically if the host on which runs could run containers using any of the supported runtimes (e.g. rkt or runc).
If so, Vdsm will advertise the capabilities to Engine. Vdsm will use the existing `additionalFeatures` capability.
To the detect the container support, Vdsm will just try to load the [bridge python module](http://github.com/mojaves/convirt),
much like it already does for glusterfs, and will depend on such module for the low-level details. Vdsm will never talk directly to the container runtime,
like it never does to emulators.

#### The feature should be completely transparent to the other flows
The main focus of oVirt is managing virtual machines. Container support will be fit in this context and framework. This means that the container support
is always additional and never hurts in any way the ability of an host to run VMs. A container-enabled host will be able to run side-by-side VMs and containers.
Inside the system, a container will be represented like a feature-reduced VMs. For example, migrations will always fail; in a later stage, Engine could
recognize the container "VMs" and just disable the features instead of allowing them and always see them fail.

#### The feature should require minimal changes to the current infrastructure
We represent the containers as dumbed down VM, in order to leverage all existing storage, monitoring and networking infrastructures.
In the Engine data model, all the information a container need already fits in the VM representation.
We want to leverage integration with existing networking, monitoring and storage subsystem. A key factor to achieve this goal is the API of
the [bridge python module](http://github.com/mojaves/convirt). This module mimics the libvirt API to cleanly fit in the existing Vdsm.

### Implementation stages

1. Run containers alongside VMs (oVirt 4.0)

2. Better integration in Engine

3. Run containers inside VMs

### Open issues

The following is a list of issues not yet settled

1. Container attributes
Some container runtime requires extra parameters (e.g. executable to run). We will use custom properties for this, but needs to be tested

2. Storage integration
Container run images which should be stored into a Storage Domain. However, oVirt, at least initially, can't provide any facility to create them.
So, container images (e.g. [AppC images](https://github.com/appc) needs to be uploaded into the system.
The fact is that those image should be put into a data domain, but we lack the facility to store them there.
Once the images are into a data domain, the existing flows should work as they do right now for plain raw images on File Storage.

3. Network integration
We want to leverage existing infrastructure. No issues yet, but this was not explored yet.

4. Monitoring integration
No Engine changes. Vdsm will report the stats as the container were VMs. Few stat could be missed, or perhaps faked.

5. Engine UI changes
We will need some UI changes on the create VM flow to select the container runtime.
The container runtime will not be editable once set
Engine will initially allow any VM operation on containers, and the actions will fail once started. See implementation stage 2 for smarter
integration

### Container runtime technologies

*   systemd-nspawn

discarded because its very man page has an early warning:

       Note that even though these security precautions are taken systemd-nspawn is not suitable for secure container setups. Many of the security features may be circumvented and are hence 
       primarily useful to avoid accidental changes to the host system from the container.
       The intended use of this program is debugging and testing as well as building of packages, distributions and software involved with boot and systems management.

*   libvirt-lxc

discarded because in <https://access.redhat.com/articles/1365153> we can read

       The following libvirt-lxc packages are deprecated starting with Red Hat Enterprise Linux 7.1:

*   docker

Summary:

*   + very popular solution
*   - partially duplicates some oVirt infrastructure
*   - overcrowded product space. Should oVirt tackle this?

Discussion: The only concerns about integrating with Docker are the duplication of effort between oVirt and Docker project; furthermore is not clear if oVirt should lean that far (at least initially) from its strong area to chime in an already overcrowded space.

*   runc

Summary:

*   + core infrastructure from docker, the "plumbing" stripped from all docker "porcelaine", as advertiesed on <http://runc.io/>
*   + tools to import/export from docker, albeit in development or planeed
*   + minimizes duplication with oVirt infrastrcture, while retaining most of the strong points of docker
*   - spec and tool still rough, not finalized
*   - more plumbing needed w.r.t docker

Discussion: runc could be the sweet spot, because it provides the strong points of docker, while allowing integration as deeper as we can and want to provide in oVirt. While the ultimate code is probably solid (being runc spun off docker), the docs, tooling and support may be scarce or still volatile.

*   rkt

Summary:

*   TO BE FILLED

Discussion: TO BE FILLED


### How to try the feature

We want to offer early access to this feature, to gather feedback as soon as possible. Just bear in mind
that the feature is in development stage, pre-technical-preview quality. Engine integration is in early
planning stage, so expect some hiccups.
We recommend a dedicated setup to try out this feature.
You will need to install Vdsm from source.

1. prepare a regular oVirt installation. One virtualization host is sufficient. Storage should be file-based:
   either shared NFS storage or local posix FS storage.

2. install `rkt' >= 1.0 on that host.
   Please refer to the [official documentation](https://coreos.com/rkt/docs/latest/) for this.
   Please note that there are packages for Fedora 24 available. We will provide spec files for Centos 7.x in the near future.
   Installation from sources is fine. Make sure 'rkt' works, maybe using [those instructions](https://github.com/coreos/rkt/blob/master/Documentation/getting-started-guide.md)

   *CAVEAT*: If you are using selinux, [you may need to disable it to run rkt containers](https://github.com/coreos/rkt/issues/1882)

3. Install the [container runtime python module](https://github.com/mojaves/convirt), codename 'convirt'.
   This is a regular python package, so it is installable using the [standard means](https://docs.python.org/2/install/).
   Availability of RPM/deb packages and uploading on [PyPI](https://pypi.python.org/pypi) is planned for near future.

4. [Fetch Vdsm from gerrit](http://www.ovirt.org/develop/developer-guide/vdsm/developers/). Please use the master branch, and
   apply [this patch series](https://gerrit.ovirt.org/#/q/topic:container-support) on top of the master branch. Don't be
   scared by patch named 'WIP' or 'HACK', they are just reminder that the posted solution isn't the final one.
   [Rebuild and install Vdsm](http://www.ovirt.org/develop/developer-guide/vdsm/developers/). Make sure to install the
   'imagerepo' hook (provided by [this patch](https://gerrit.ovirt.org/#/c/54873/) you should already applied).

5. Make sure Vdsm works. Try running

     # vdsClient -s 0 getVdsCaps

   from the host on which you reinstalled the patched Vdsm. Make sure you see the following in the output

     additionalFeatures = [{'containers': ['rkt']}]

   If you can see this, congrats! your Vdsm can run containers

6. Prepare the image repository on the virtualization host. This is needed because you can't yet upload
   externally build images on Storage Data Domain. Any directory on the virtualization host is fine, make
   sure is writable by `vdsm:kvm`. Upload there all the images you want to use.

   Example:

     /srv/container/images

7. You can use any 3.6.x Engine to try this feature. You just need to define few custom properties:

     # engine-config -s UserDefinedVMProperties='imagerepo=^/srv/convirt/data$;imagename=^hello-0.0.1-linux-amd64.aci$;container=^(rkt|runc)$' --cver=3.6

8. Define one or more Vms, to be used as decoys for containers. This is needed because Engine hasn't yet gained
   explicit supports for containers, so it knows only about VMs.

   *CAVEAT*: You don't need a disk for each Vm (container support will ignore it and use images
   see below), but Engine won't let you start a Vm without a boot device. You may want to use PXE boot to workaround
   around this. The container runtime will ignore this setting, is just to pass Engine's check

9. In the phony VMs you defined, make sure you enable the two custom variables you created in step #8

    * `imagerepo` should be set to the path you prepared on the host in step #6 (example: `/srv/container/images`)
    * `imagename` full name of the container you want to run. The corresponding image must be present into `imagerepo` directory on the virtualization host.
    * `container` should be set to `rkt` (no other container runtime are supported yet)

10. You can now run the container! just pick it from the VM listing and run it.
   Containers can't migrate or be hibernated/restored, and monitoring is not yet supported.


### Patches/code

*  [container python module](http://github.com/mojaves/convirt)
*  [Vdsm patches](https://gerrit.ovirt.org/#/q/status:open+project:vdsm+branch:master+topic:container-support)
*  Engine patches: still pending


### Documentation / External references

*   [systemd-nspawn](http://www.freedesktop.org/software/systemd/man/systemd-nspawn.html)
*   [libvirt-lxc](https://libvirt.org/drvlxc.html)
*   [opencontainers (runc)](https://github.com/opencontainers)
*   [docker](https://www.docker.com/)

### Testing

TODO

### Contingency Plan

We add a new optional feature, so there is no negative fallback and no contingency plan, oVirt will just keep working as usual.

### Release Notes

### Comments and Discussion

*   Refer to [Talk:Your feature name](Talk:Your feature name)

<Category:Feature>
