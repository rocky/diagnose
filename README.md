# Probes

A library for instrumenting Python code at runtime.

Structured logs and metrics and observability are great, but almost always
require you to alter your code, which interrupts the flow when reading code.
It also typically requires a build cycle to alter; it's no fun making a ticket,
getting review, waiting for a build and deploy, and then doing it all again
to back out your temporary additions. This is doubly true when doing research,
where you might perform a dozen small experiments to measure your live code.

This library allows you to dynamically add probes at runtime instead.
Probes are:
* reliable: errors will never affect your production code
* ephemeral: set a "lifespan" (in minutes), after which point the probe detaches
* comprehensive: all references to the target function are instrumented
* fast: measure most functions with fast local lookups; uses hunter (in Cython) for more invasive internal probes.

Individual probes can be created directly with the FunctionProbe class:

```#python
>>> from path.to.module import myclass
>>> myclass().add13(arg=5)
18
>>> p = probes.FunctionProbe("path.to.module.myclass.add13")
>>> p.instruments["foo"] = probes.LogInstrument("foo", "arg", internal=False)
>>> p.start()
>>> myclass().add13(arg=5)
Probe (foo) = 5
18
```

Managers
--------

In a running system, we want to add, remove, start, and stop probes without
having to code at an interactive prompt or restart the system; we do this
with a ProbeManager. Start by configuring the global probes.manager:

```#python
>>> probes.manager.instrument_classes = {
    "log": LogInstrument,
    "hist": MyHistogramInstrument,
    "incr": MyIncrementInstrument,
}
>>> probes.manager.global_namespace.update({"foo": foo})
```

Later, you can define probes:

```#python
>>> probes.manager.specs["probe-1"] = {
    "target": "myapp.module.file.class.method",
    "instrument": {
        "type": "log",
        "name": "myapp.method",
        "value": "result",
        "internal": False,
        "custom": {},
    },
    "lifespan": 10,
    "lastmodified": datetime.datetime.utcnow(),
    "applied": {},
}
```

Then call `probes.manager.apply()`, either when you add a probe, or on a
schedule if your store is in MongoDB and the process defining probes is not
the target process.

The `applied` dictionary will be filled with information about which processes
have applied the probe, and whether they encountered any errors.
