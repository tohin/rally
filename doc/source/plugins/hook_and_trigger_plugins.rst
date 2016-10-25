..
      Copyright 2016 Mirantis Inc. All Rights Reserved.

      Licensed under the Apache License, Version 2.0 (the "License"); you may
      not use this file except in compliance with the License. You may obtain
      a copy of the License at

          http://www.apache.org/licenses/LICENSE-2.0

      Unless required by applicable law or agreed to in writing, software
      distributed under the License is distributed on an "AS IS" BASIS, WITHOUT
      WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the
      License for the specific language governing permissions and limitations
      under the License.

.. _plugins_hook_and_trigger_plugins:


Hook as a plugin
================

Hook plugins allow to run some code separately from main runner, because they
are implemented as threads and executed in parallel.
For example hooks can be used to make a destructive action on HA cluster on
specified iteration.

Let's create a simple hook plugin that runs provided command from shell.

Creation
^^^^^^^^

Inherit a class for your plugin from the base *Hook* class and implement its API (the *run()* method):

.. code-block:: python

    import shlex
    import subprocess

    from rally import consts
    from rally.task import hook


    @hook.configure(name="sys_call")
    class SysCallHook(hook.Hook):
        """Performs system call."""

        CONFIG_SCHEMA = {
            "$schema": consts.JSON_SCHEMA,
            "type": "string",
        }

        def run(self):
            proc = subprocess.Popen(shlex.split(self.config),
                                    stdout=subprocess.PIPE,
                                    stderr=subprocess.STDOUT)
            proc.wait()
            if proc.returncode:
                self.set_error(
                    exception_name="n/a",  # no exception class
                    description="Subprocess returned {}".format(proc.returncode),
                    details=proc.stdout.read(),
                )


Trigger plugin
^^^^^^^^^^^^^^

In order to use the hook we need to specify iteration on which the hook
should be executed. Trigger plugins created to provide a flexible way to
specify when and how many times the hook should be executed.

Let's take a look at simple trigger plugin that runs hook on specified list of
iterations:

.. code-block:: python

    from rally import consts
    from rally.task import trigger


    @trigger.configure(name="event")
    class EventTrigger(trigger.Trigger):
        """Triggers hook on specified event and list of values."""

        CONFIG_SCHEMA = {
            "type": "object",
            "$schema": consts.JSON_SCHEMA,
            "oneOf": [
                {
                    "properties": {
                        "unit": {"enum": ["time"]},
                        "at": {
                            "type": "array",
                            "minItems": 1,
                            "uniqueItems": True,
                            "items": {
                                "type": "integer",
                                "minimum": 0,
                            }
                        },
                    },
                    "required": ["unit", "at"],
                    "additionalProperties": False,
                },
                {
                    "properties": {
                        "unit": {"enum": ["iteration"]},
                        "at": {
                            "type": "array",
                            "minItems": 1,
                            "uniqueItems": True,
                            "items": {
                                "type": "integer",
                                "minimum": 1,
                            }
                        },
                    },
                    "required": ["unit", "at"],
                    "additionalProperties": False,
                },
            ]
        }

        def get_listening_event(self):
            return self.config["unit"]

        def on_event(self, event_type, value=None):
            if not (event_type == self.get_listening_event()
                    and value in self.config["at"]):
                # do nothing
                return
            super(EventTrigger, self).on_event(event_type, value)


Usage
^^^^^

You can refer to your Hook in the benchmark task configuration files:

.. code-block:: json

    {
        "Dummy.dummy": [
            {
                "args": {
                    "sleep": 0.01
                },
                "runner": {
                    "type": "constant",
                    "times": 1500,
                    "concurrency": 1
                },
                "context": {
                    "users": {
                        "tenants": 1,
                        "users_per_tenant": 1
                    }
                },
                "hooks": [
                    {
                        "name": "sys_call",
                        "args": "/bin/echo 123",
                        "trigger": {
                            "name": "event",
                            "args": {
                                "unit": "iteration",
                                "at": [5, 50, 200, 1000]
                            }
                        }
                    }
                ]

            }
        ]
    }
