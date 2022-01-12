
.. _minio-restore-hardware-failure:

==============================
Recover after Hardware Failure
==============================

.. default-domain:: minio

.. contents:: Table of Contents
   :local:
   :depth: 1

Distributed MinIO deployments rely on :ref:`Erasure Coding
<minio-erasure-coding>` to provides built-in tolerance for multiple disk or node
failures. Depending on the deployment topology, MinIO can tolerate the loss of
up to half the drives or nodes in the deployment while maintaining read access
to stored objects.

While MinIO can operate in a degraded state without significant performance
loss, administrators should replace failed hardware at the earliest convenience
to mitigate the risk of more significant failures.

The following table lists the typical types of failure in a MinIO deployment
and links to procedures for recovering from each:

.. list-table::
   :header-rows: 1
   :widths: 30 70
   :width: 100%

   * - Failure Type
     - Description

   * - :ref:`Drive Failure <minio-restore-hardware-failure-drive>`
     - MinIO supports hot-swapping failed drives with new healthy drives. 

   * - :ref:`Node Failure <minio-restore-hardware-failure-node>`
     - MinIO detects when a node rejoins the deployment and begins aggressively
       healing data previously stored on that node.

.. admonition:: MinIO Professional Support
   :class: note

   |subnet| provides 24/7 direct-to-engineering support for MinIO deployments.
   SUBNET is a key component for ensuring smooth and reliable operation of MinIO
   deployments, including root-cause analysis, health diagnostics, and
   architecture reviews.

   Existing SUBNET users should `log in <https://subnet.min.io/>`__ and
   create a new issue to synchronize any recovery operations with MinIO
   engineering.

   Community users can seek support on the `MinIO Community Slack
   <https://minio.slack.com>`__. Community Support is best-effort only and has
   no SLAs around responsiveness.

.. _minio-restore-hardware-failure-drive:

Drive Failure Recovery
----------------------

MinIO supports hot-swapping failed drives with new healthy drives. MinIO
detects and aggressively heals those drives without performing any restart
procedure.

The procedure can be summarized as follows:

1. Unmount the failed drive.
2. Replace the failed drive with a new drive of the same type and capacity.
3. Format the drive to match the filesystem (e.g. xfs) and set a matching label.
4. Update ``/etc/fstab`` (if necessary) and remount the drive.
5. Monitor MinIO logs to verify detection and healing of the drive.
6. Use :mc-cmd:`mc admin heal` to inspect healing status.

Replacement drives should be freshly formatted and empty. MinIO healing ensures
consistency and correctness of all data restored onto the drive. **Do not**
attempt to manually recover or migrate data from the failed drive onto the new
healthy drive.

The replacement drive hardware should be substantially similar to the failed
drive. For example, replace a failed SSD with another SSD drive of the same
capacity. While you can use drives with larger capacity, MinIO uses the
*smallest* drive's capacity as the ceiling for all drives in the :ref:`Server
Pool <minio-intro-server-pool>`.

The following steps provide a more detailed walkthrough of drive replacement.
These steps assume a MinIO deployment where each node manages drives using
``/etc/fstab`` with per-drive labels as per the :ref:`documented prerequisites <minio-installation>`.

1) Unmount the failed drive(s)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Unmount each failed drive using ``umount``. For example, the following
command unmounts the drive at ``/dev/sdb``:

.. code-block:: shell

   umount /dev/sdb

2) Replace the failed drive(s)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Replace each failed drive with a new known-healthy drive of similar type. For
example, replace an HDD with an HDD, SSD with SSD, and NVMe with NVMe. 

- The capacity of the drive should *at minimum* match the replaced
  drive. MinIO does not use any extra capacity on the drive beyond that of
  other drives in the :ref:`Server Pool <minio-intro-server-pool>`. 

- The speed of the drive should *at minimum* match the replaced drive. Using
  a slower drive may result in unexpected performance issues.

3) Format the new drive(s)
~~~~~~~~~~~~~~~~~~~~~~~~~~

Format each new drive to match the other drives on the node (e.g. ``xfs``). 
Specify a matching label for the replaced drive *if* using label-based
drive management:

.. code-block:: shell

   mkfs.xfs /dev/sdb -L DISK1

MinIO **strongly recommends** using label-based mounting to ensure consistent
drive order that persists through system restarts.

4) Review and Update ``fstab``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Review the ``/etc/fstab`` file and update as needed such that the entry for
the failed disk points to the newly formatted replacement.

- If using label-based disk assignment, ensure that each label points to the
  correct newly formatted disk.

- If using UUID-based disk assignment, update the UUID for each point based on
  the newly formatted disk. Use ``lsblk`` to view disk UUIDs.

For example, consider 

.. code-block:: shell

   $ nano /etc/fstab

     # <file system>  <mount point>  <type>  <options>         <dump>  <pass>
     LABEL=DISK1      /mnt/disk1     xfs     defaults,noatime  0       2
     LABEL=DISK2      /mnt/disk2     xfs     defaults,noatime  0       2
     LABEL=DISK3      /mnt/disk3     xfs     defaults,noatime  0       2
     LABEL=DISK4      /mnt/disk4     xfs     defaults,noatime  0       2

No further change to this ``fstab`` are required since the replace disk
at ``/mnt/disk1`` uses the same label ``DISK1``.

5) Remount the Replaced Drive(s)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Use ``mount -a`` to remount the drives unmounted at the beginning of this
procedure:

.. code-block:: shell
   :class: copyable

   mount -a

The command should result in remounting of all of the replaced drives.

6) Monitor MinIO for Drive Detection and Healing Status
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Use :mc-cmd:`mc admin console` command *or* ``journalctl -u minio`` for
``systemd``-managed installations to monitor the server log output after remounting
drives. The output should include messages identifying each formatted and empty
drive.

Use :mc-cmd:`mc admin heal` to monitor the overall healing status on the
deployment. MinIO aggressively heals replaced drive(s) to ensure rapid recovery
from the degraded state.

7) Next Steps
~~~~~~~~~~~~~

Monitor the cluster for any further drive failures. Some drive batches may fail
in close proximity to each other. Deployments seeing higher than expected drive
failure rates should schedule dedicated maintenance around replacing the known
bad batch. Consider using |subnet| to coordinate with MinIO engineering around
guidance for any such operations.

.. _minio-restore-hardware-failure-node:

Node Failure Recovery
---------------------

If a MinIO node suffers complete hardware failure (e.g. loss of all drives,
data, etc.), the node begins healing operations once it rejoins the deployment.

The procedure can be summarized as follows:

1. Bring up the replacement node, ensuring it meets or exceeds the configuration
   of the failed node.
2. Update DNS to ensure the hostname for the failed node points at the
   replacement node.
3. Start the MinIO process on the new node ensuring the settings match all other
   nodes in the deployment. The MinIO binary **must** match across all nodes.
4. Check MinIO logs to verify the node has reconnected.
5. Use :mc-cmd:`mc admin heal` to confirm the healing status.

The replacement node hardware should be substantially similar to the failed
node. There are no negative performance implications to using improved compute
hardware.

The replacement drive hardware should be substantially similar to the failed
drive. For example, replace a failed SSD with another SSD drive of the same
capacity. While you can use drives with larger capacity, MinIO uses the
*smallest* drive's capacity as the ceiling for all drives in the :ref:`Server
Pool <minio-intro-server-pool>`.

The following steps provide a more detailed walkthrough of node replacement.
These steps assume a MinIO deployment where each node has a DNS hostname 
as per the :ref:`documented prerequisites <minio-installation>`.

1) Start up the new node hardware
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Ensure the new node has received all necessary security, firmware, and OS
updates as per industry, regulatory, or organizational standards and
requirements.

The new node software configuration should closely match that of the other
nodes in the deployment, including but not limited to the OS and Kernel 
versions and configurations. Heterogeneous software configurations may result
in unexpected or undesired behavior in the deployment.

2) Update DNS to for the new host
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Update the DNS entry for the failed host machine to point to the new host.

For example, if the previous host was reachable at
``https://minio-1.example.net``, ensure the new host is now reachable at that
hostname.

3) Download and Prepare the MinIO Server
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Follow the :ref:`deployment procedure <minio-installation>` to download
and run the MinIO server using a matching configuration as all other nodes
in the deployment.

- The MinIO server version *must* match across all nodes
- The MinIO service and environment file configurations *must* match across
  all nodes.

4) Rejoin the node to the deployment
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Start the MinIO server process on the node and monitor the process output
using :mc-cmd:`mc admin console` or by monitoring the MinIO service logs
using ``journalctl -u minio`` for ``systemd`` managed installations.

The server output should indicate that it has detected the other nodes
in the deployment and begun healing operations.

Use :mc-cmd:`mc admin heal` to monitor overall healing status on the
deployment. MinIO aggressively heals the node to ensure rapid recovery
from the degraded state.

5) Next Steps
~~~~~~~~~~~~~

Continue monitoring the deployment until healing completes. Deployments with
persistent and repeated node failures should schedule dedicated maintenance to
identify the root cause. Consider using |subnet| to coordinate with MinIO
engineering around guidance for any such operations.
