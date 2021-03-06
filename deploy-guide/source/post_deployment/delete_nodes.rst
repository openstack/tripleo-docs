.. _delete_nodes:

Deleting Overcloud Nodes
========================

You can delete specific nodes from an overcloud with command::

    openstack overcloud node delete --stack $STACK_NAME --templates [templates dir] <list of nova instance IDs or hostnames starting in Train>

This command updates the heat stack with updated numbers and list of resource
IDs (which represent nodes) to be deleted.

.. admonition:: Train
   :class: train

   In Train, we added a user confirmation to the scale down command to
   prevent accidental node removal.
   To skip it, please use "--yes".

.. admonition:: Train
   :class: train

   Starting in Train and onward, `openstack overcloud node delete` can take
   a list of server hostnames instead of instance ids. However they can't be
   mixed while running the command. Example: if you use hostnames, it would
   have to be for all the nodes to delete.

.. note::
  If you are :ref:`baremetal_provision` then follow those
  scale-down instructions to call ``openstack overcloud node delete`` with a
  ``--baremetal-deployment`` argument instead of passing a list of nodes to
  delete as arguments.

.. note::
   If you passed any extra environment files when you created the overcloud (for
   instance, in order to configure :doc:`network isolation
   <../features/network_isolation>`), you must pass them again here
   using the ``-e`` or ``--environment-file`` option to avoid making undesired
   changes to the overcloud.

.. note::
   Before deleting a compute node or a cephstorage node, please make sure that
   the node is quiesced, see :ref:`quiesce_compute` or
   :ref:`quiesce_cephstorage`.

.. note::
   A list of nova instance IDs can be listed with command::

       openstack server list
