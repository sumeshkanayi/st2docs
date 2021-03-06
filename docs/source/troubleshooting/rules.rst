Rule is not Being Matched
=========================

You created a rule which supposed to fire an action when a particular trigger
is emitted, but for some reason, this action is not being called.

There are multiple reasons why this could be the case. If you end up in such a situation,
follow this debugging procedure:

1. Validate if the rule was enforced
2. Validate if a trigger was emitted
3. Validate rules are being loaded for the trigger
4. Check rule criteria matches the emitted trigger
5. Action runner service is running
6. Ask for help!

1. Validate If Rule Was Enforced
--------------------------------

You can use the CLI to list all enforcements for a rule, using this command:

.. code-block:: bash

    st2 rule-enforcement list

To filter rule enforcements by rule:

.. code-block:: bash

    st2 rule-enforcement list --rule=git.st2.webhook.github.pulls.merge.sample

    +--------------------------+----------------------+----------------------+----------------------+----------------------+
    | id                       | rule_ref             | trigger_instance_id  | execution_id         | enforced_at          |
    +--------------------------+----------------------+----------------------+----------------------+----------------------+
    | 5661ebe932ed351812c5b0ac | git.st2.webhook.gith | 5661ebe832ed351812c5 | 5661ebe932ed351812c5 | Fri, 04 Dec 2015     |
    |                          | ub.pulls.merge.sampl | b0a8                 | b0ab                 | 19:39:21 UTC         |
    |                          | e                    |                      |                      |                      |
    | 5661ebea32ed351812c5b0b2 | git.st2.webhook.gith | 5661ebea32ed351812c5 | 5661ebea32ed351812c5 | Fri, 04 Dec 2015     |
    |                          | ub.pulls.merge.sampl | b0ae                 | b0b1                 | 19:39:22 UTC         |
    |                          | e                    |                      |                      |                      |
    | 5661ebeb32ed351812c5b0ba | git.st2.webhook.gith | 5661ebeb32ed351812c5 | 5661ebeb32ed351812c5 | Fri, 04 Dec 2015     |
    |                          | ub.pulls.merge.sampl | b0b6                 | b0b9                 | 19:39:23 UTC         |
    |                          | e                    |                      |                      |                      |
    +--------------------------+----------------------+----------------------+----------------------+----------------------+

As you can see, you'll see the trigger instance and the execution for a rule enforcement if
there were any. If no execution was kicked off, there won't be an execution id. This means
st2 had issues running the action (invalid parameters, ``action_ref`` not present etc). You'll need
to check the rules engine logs for exceptions. You can ``grep`` using trigger instance id.
If you don't see an enforcement, check if there was a trigger emitted using following section.

2. Validate Trigger is Emitted
-------------------------------

Here, you check if a trigger is being emitted and flowing to the rules engine. The easiest way to
do that is to use the CLI. This command will list all the trigger instances ever seen in the
system:

.. code-block:: bash

    st2 trigger-instance list

You can also filter trigger instances by trigger reference:

.. code-block:: bash

    st2 trigger-instance list --trigger=test_pack.test_trigger

You can also filter trigger instances by timestamp using ``timestamp-lt`` and ``timestamp-gt`` flags:

.. code-block:: bash

    st2 trigger-instance list --trigger=test_pack.test_trigger --timestamp-lt="2015-12-04T12:00:01.000000Z" --timestamp-gt="2015-12-03T12:00:01.000000Z"

It is possible that a trigger-instance is present in the list but it does not have a status
``processed``. This will also tell you that the RulesEngine tried to process the TriggerInstance
but failed. Other possible states are ``processing`` and ``pending``.

.. code-block:: bash

    $ st2 trigger-instance get 57228f31d9d7ed0becb34e06
    +-----------------+---------------------------------------------------------+
    | Property        | Value                                                   |
    +-----------------+---------------------------------------------------------+
    | id              | 57228f31d9d7ed0becb34e06                                |
    | trigger         | test_pack.test_trigger                                  |
    | occurrence_time | 2016-04-28T22:31:13.913000Z                             |
    | payload         | {                                                       |
    |                 |     "executed_at": "2016-04-28 22:31:13.910217+00:00",  |
    |                 |     "schedule": null                                    |
    |                 | }                                                       |
    | status          | processing_failed                                       |
    +-----------------+---------------------------------------------------------+

If you don't see any trigger instances for your trigger, then check the sensor container logs.
You can check the sensor container service log (``/var/log/st2/st2sensorcontainer.*.log``)
to see if triggers are emitted.

Keep in mind that this approach only works for triggers being emitted by sensors and won't work for
triggers generated by incoming webhooks and triggers generated by timers.

If you don't see this line, this means that trigger is not being emitted and flowing to the rules
engine. This could either mean that the sensor is misconfigured or not running or that there is
some other issue with the sensor.

If you don't see a trigger emitted, fix the sensor. If you do see one, then it's time to check
the rules engine.

3. Validate If Rules for a Trigger Are Being Loaded
---------------------------------------------------

First begin by looking at the loaded rules and validate that there are rules that apply
to the Trigger.

The following command will list all the rules for a specific trigger:

.. code-block:: bash

    st2 rule list --trigger=test_pack.test_trigger

If there are rules in this list then it tells us that there are rules in the system that actually
match a given Trigger.

In case nothing is returned and the result is surprising then it is worth looking into the rules
engine logs. By default, rules engine logs are stored in the ``/var/log/st2/st2rulesengine.log``
file. You should inspect this file (``cat``, ``grep`` and ``tail`` are your friends) and look for
a line similar to this:

.. code-block:: bash

    2015-02-23 15:13:51,250 INFO [-] Found <n> rules defined for trigger <trigger name>

For example:

.. code-block:: bash

    2015-02-23 15:13:51,250 INFO [-] Found 1 rules defined for trigger st2.generic.actiontrigger

If you don't see any rules being loaded, there is a mismatch in the rule definition w.r.t trigger.
See :ref:`st2-rule-tester<ref-rule-tester-post-mortem-debug>` usage for this specific case to see
how to confirm the mismatch failure between rule and triggerinstance. ``st2-rule-tester`` will be
able to validate both ``trigger ref`` and rule criteria.

If this does not work then you can also use the CLI to check the rule and validate the 
``trigger ref`` is indeed right by visual inspection:

.. code-block:: bash

    st2 rule get test_pack.test_rule

If you validated that rules are being loaded, it's time to validate rule criteria.

4. Verifying the Rule Criteria
------------------------------

Now that we have established that trigger is indeed being emitted, we are going to verify the rule
criteria.

We are again going to resort to the rules engine service logs since this is the place where the
rules are matched against triggers and evaluated.

If your criteria matches the emitted trigger, you should see a message similar to this:

.. code-block:: bash

    2015-02-23 15:24:11,324 INFO [-] Matched 1 rule(s) for trigger_instance st2.generic.actiontrigger

If the message says ``Matched 0 rule(s)`` this means that the emitted trigger doesn't match the
defined rule criteria. Usually this is simply a feature - the trigger which was emitted is not the
one you are interested in. For this reason, an enforcement object is not written to db when rule
criteria doesn't match incoming payload.  If you believe the rule should indeed match the defined
trigger, the next step is to debug rule criteria and make sure it's configured correctly.

For information on how to use the ``st2-rule-tester`` tool to do that, please refer
to the :ref:`testing-rules` section.

If you see rule criteria matched and there is an enforcement object, but no execution was kicked
off, then validate if action runners are up and running.

5. Verifying That Action Runner Service Is Running
---------------------------------------------------

You have now fixed your your rule criteria (or there was nothing wrong with it), but for some
reason action is still not being fired/executed.

If you see a message similar to the one below in your rules engine service log, this mean that
everything is indeed working as it should - trigger is successfully matched against the rule
criteria and an action execution is scheduled.

.. code-block:: bash

    2015-02-23 15:32:21,694 INFO [-] Invoking action core.local for trigger_instance 54eb48050640fd32c2d34034 with data {"cmd": "echo \"2015-02-23 15:32:21.663471\""}.
    2015-02-23 15:32:21,788 AUDIT [-] Action execution scheduled. LiveAction=LiveActionDB(action="core.local", ...

If you see this message, but the action is still not being executed, this usually simply means
that the action runner, the service which is responsible for running actions is not running.

The easiest way to check if the service is running is to use ``st2ctl``:

.. code-block:: bash

    sudo st2ctl status

If the service is for some reason not running, you can use the ``sudo st2ctl start`` command to try
to start all the services again. If the service still doesn't start after running the command, this
usually means a configuration error (e.g. invalid database information or credentials). The best
way to debug that is to look into the action runner service logs -
``/var/log/st2/logs/st2actionrunner.*.log``.

6. Ask for Help!
----------------

You have exhausted self help directions. Contact us using the :ref:`ask for help<ref-ask-for-help>`
section. Please have the output of
``st2 rule-enforcement list --rule=<rule_being_debugged>``,
``st2 trigger-instance list --trigger=<trigger>`` and rule YAML ready so we can help you debug
faster!
