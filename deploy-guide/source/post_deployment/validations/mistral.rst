Running validations using mistral
---------------------------------

Running single validations
^^^^^^^^^^^^^^^^^^^^^^^^^^

If you want to run one validation in particular (because you're trying
things out or want to see whether you fixed the failure), you can run
it like so::

   $ source stackrc
   $ openstack workflow execution create tripleo.validations.v1.run_validation '{"validation_name": "undercloud-ram"}'

This will run the "undercloud-ram.yaml" validation.

The undercloud comes with a set of default validations, which are stored in a
Swift container called ``tripleo-validations``. To get the list of available
validations, run ``openstack container list tripleo-validations``. To get the
name of the validation (``undercloud-ram`` here) find the right one there and
use its filename without an extension.

The command will return an execution ID, which you can query for the
results. To find out whether the validation finished, run::

  $ openstack workflow execution show ID

Note the ``State`` value: it can be either ``RUNNING`` or ``SUCCESS``.
Note that success here only means that the validation finished, not
that it succeeded. To find that out, we need to read its output::

  $ mistral execution-get-output ID | jq .stdout -r

.. note:: There is an ``openstack workflow execution show output``
          command which should do the same thing, but it currently
          doesn't work in all environments supported by |project|.

This will return the hosts the validation ran against, any warnings
and error messages it encountered along the way as well as an overall
summary.


Custom validations
^^^^^^^^^^^^^^^^^^

Support for `custom validations`_ has been added in the Rocky development cycle.
It allows operators to add their own bespoke validations, in cases when it's
not appropriate to include these in the set of default TripleO validations.

Custom validations are associated with deployment plans and stored in the
plan's Swift container, together with the rest of the plan files.

To add custom validations for a deployment plan, create a ``custom-validations``
subdirectory within the deployment plan Swift container and place the
validations yaml files there.

To run a custom validation, follow the same procedure as for running one of the
default validations - determine the name of the validation by listing the contents
of the ``custom-validations`` subdirectory, and supply that name (without the
.yaml extension) to the ``run_validation`` workflow.

If a validation with the same name is found both in the set of default
validations, and in custom validations, the custom validation is always picked.
This means that, if you wish to override a default validation with your custom
implementation of it, all you need to do is create a validation with the same
name and place it in the ``custom-validations`` subdirectory of the Swift
container holding the deployment plan.


.. _running_validation_group:

Running a group of validations
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The deployment documentation highlights places where you can run a
specific group of validations. Here's how to run the
``pre-deployment`` validations::

  $ source stackrc
  $ openstack workflow execution create tripleo.validations.v1.run_groups '{"group_names": ["pre-deployment"]}'

Unfortunately, getting the results of all these validations is more
complicated than in the case a single one. You need to see the tasks
the workflow above generated and see their results one by one: ::

  $ openstack task execution list

Look for tasks with the ``run_validation`` name. Then take the ID of
each of those and run::

  $ mistral task-get-result ID | jq .stdout -r

.. note:: There is a ``task execution show result`` command that
          should do the same thing, but it's not working on all
          platforms supported by |project|.

Since this can be tedious and hard to script, you can get the list of
validations belonging to a group instead and run them one-by-one using
the method above::

  $ openstack action execution run tripleo.validations.list_validations '{"groups": ["pre-deployment"]}' | jq ".result[] | .id"

Another example are the "pre-upgrade" validations which are added during the P
development cycle. These can be executed as
the example above but instead using the "pre-upgrade" group::

    openstack workflow execution create tripleo.validations.v1.run_groups '{"group_names": ["pre-upgrade"]}'

    +-------------------+--------------------------------------+
    | Field             | Value                                |
    +-------------------+--------------------------------------+
    | ID                | 3f94a17b-835b-4a82-93af-a6cddd676ed8 |
    | Workflow ID       | e211099f-2c9b-46cd-a536-e38595ae8e7f |
    | Workflow name     | tripleo.validations.v1.run_groups    |
    | Description       |                                      |
    | Task Execution ID | <none>                               |
    | State             | RUNNING                              |
    | State info        | None                                 |
    | Created at        | 2017-06-29 12:01:35                  |
    | Updated at        | 2017-06-29 12:01:35                  |
    +-------------------+--------------------------------------+

You can monitor the progress of the execution by getting its status and also
output::

    mistral execution-get $ID
    mistral execution-get-output $ID

When any of the validations fail the execution will have a ERROR status.
You can query the individual validations in the group to determine the exact
reasons that the validation fails. For example::

        for i in $(mistral execution-list | grep tripleo.validations.*ERROR | awk '{print $2}'); do mistral execution-get-output $i; done
    {
        "result": "Failure caused by error in tasks: get_servers\n\n  get_servers [task_ex_id=a6ef7d32-4678-4a58-85fe-bf2da8a963ae] -> Failed to run action [action_ex_id=3a9a81e2-d6b0-4380-8985-41d6f4e18f3a, action_cls='<class 'mistral.actions.action_factory.NovaAction'>', attributes='{u'client_method_name': u'servers.list'}', params='{}']\n NovaAction.servers.list failed: <class 'keystoneauth1.exceptions.connection.ConnectFailure'>: Unable to establish connection to http://192.168.24.1:8774/v2.1/servers/detail: ('Connection aborted.', BadStatusLine(\"''\",))\n    [action_ex_id=3a9a81e2-d6b0-4380-8985-41d6f4e18f3a, idx=0]: Failed to run action [action_ex_id=3a9a81e2-d6b0-4380-8985-41d6f4e18f3a, action_cls='<class 'mistral.actions.action_factory.NovaAction'>', attributes='{u'client_method_name': u'servers.list'}', params='{}']\n NovaAction.servers.list failed: <class 'keystoneauth1.exceptions.connection.ConnectFailure'>: Unable to establish connection to http://192.168.24.1:8774/v2.1/servers/detail: ('Connection aborted.', BadStatusLine(\"''\",))\n"
    }
    {
        "status": "FAILED",
        "result": null,
        "stderr": "",
        "stdout": "Task 'Fail if services were not running' failed:\nHost: localhost\nMessage: One of the undercloud services was not active. Please check openstack-heat-api first and then confirm the status of undercloud services in general before attempting to update or upgrade the environment.\n\nTask 'Fail if services were not running' failed:\nHost: localhost\nMessage: One of the undercloud services was not active. Please check openstack-ironic-api first and then confirm the status of undercloud services in general before attempting to update or upgrade the environment.\n\nTask 'Fail if services were not running' failed:\nHost: localhost\nMessage: One of the undercloud services was not active. Please check openstack-zaqar first and then confirm the status of undercloud services in general before attempting to update or upgrade the environment.\n\nTask 'Fail if services were not running' failed:\nHost: localhost\nMessage: One of the undercloud services was not active. Please check openstack-glance-api first and then confirm the status of undercloud services in general before attempting to update or upgrade the environment.\n\nTask 'Fail if services were not running' failed:\nHost: localhost\nMessage: One of the undercloud services was not active. Please check openstack-glance-api first and then confirm the status of undercloud services in general before attempting to update or upgrade the environment.\n\nFailure! The validation failed for all hosts:\n* localhost\n"
    }
