.. currentmodule:: ophyd

==============
 Architecture
==============

Hardware abstraction
====================


``Ophyd`` is the hardware abstraction layer that provides a consistent
interface between the underlying control communication protocol and
`bluesky <https://nsls-ii.github.io/bluesky>`_.  This is done by
bundling sets of the underlying process variables into hierarchical
devices and exposing a semantic API in terms of control system
primitives.  Two terms that will be used throughout are


  **Signal**

    Represents an atomic 'process variable'. This is nominally a
    'scalar' value and cannot be decomposed any further by layers
    above :mod:`ophyd`.  In this context an array (waveform) or string
    would be a scalar because there is no ophyd API to read only part
    of it.

  **Device**

    Hierarchy composed of Signals and other Devices.  The components
    of a Device can be introspected by layers above :mod:`ophyd` and
    may be decomposed to, ultimately, the underlying Signals.


Put another way, if a hierarchical device is a tree, **Signals** are the leaves
and **Devices** are the nodes.

.. _hl_api:

Uniform High-level Interface
============================

All ophyd objects implemented a small set of methods which are used by
`bluesky`_ plans.  It is the responsibility of the `ophyd` objects to
correctly implement these methods in terms of the underlying control
system.


Read-able Interface
-------------------

The minimum set of methods an object must implement is

.. autosummary::
   :toctree: _as_gen

   ~device.BlueskyInterface.trigger
   ~device.BlueskyInterface.read
   ~device.BlueskyInterface.describe

along with three properties:

.. autosummary::
   :toctree: _as_gen

   ~ophydobj.OphydObject.name
   ~ophydobj.OphydObject.parent
   ~ophydobj.OphydObject.root


There are two optional methods which plans may use to 'enable' or
'disable' a device for data collection.  For example, a beam position
monitor maybe in continuous mode when not collecting data but be
stitched to a triggered mode for scanning.  By convention ``unstage``
'undoes' whatever ``stage`` did to the state of the underlying
hardware and should return it to the state it was before ``stage`` was
called.


.. autosummary::
   :toctree: _as_gen

   ~device.BlueskyInterface.stage
   ~device.BlueskyInterface.unstage

Two additional optional methods are used to notify devices if,
during a scan, the run is suspended.  The semantics of these methods
is coupled to :class:`~bluesky.run_engine.RunEngine`.

.. autosummary::
   :toctree: _as_gen

   ~device.BlueskyInterface.pause
   ~device.BlueskyInterface.resume

Set-able Interface
------------------

Of course, most interesting uses of hardware requires telling it to do
rather than just reading from it!  To do that the high-level API has
the ``set`` method and a corresponding ``stop`` method to halt motion
before it is complete.

The ``set`` method which returns `Status` that can be used to tell
when motion is done. It is the responsibility of the `ophyd` objects
to implement this functionality in terms of the underlying control
system. Thus, from the perspective of the `bluesky`_, a motor, a
temperature controller, a gate valve, and software pseudo-positioner
can all be treated the same.


.. autosummary::
   :toctree: _as_gen

   ~positioner.PositionerBase.set
   ~positioner.PositionerBase.stop


Configuration
-------------

In addition to values we will want to read, as 'data', or set, as a
'position', there tend to be many values associated with the
configuration of hardware.  This is things like the velocity of a
motor, the PID loop parameters of a feedback loop, or the chip
temperature of a detector.  In general these are measurements that are
not directly related to the measurement of interest, but maybe needed for
understanding the measured data.

.. autosummary::
   :toctree: _as_gen

   ~device.Device.configure
   ~device.Device.read_configuration
   ~device.Device.describe_configuration



Fly-able Interface
------------------

There is some hardware where instead of the fine-grained control
provided by ``set``, ``trigger``, and ``read`` we just want to tell it
"Go!" and check back later when it is done.  This is typically done
when there needs to coordinated motion or triggering at rates beyond
what can reasonably done in via EPICS/Python and tend to be called 'fly scans'.

The flyable interface provides four methods

.. autosummary::
   :toctree: _as_gen

   ~flyers.FlyerInterface.kickoff
   ~flyers.FlyerInterface.complete
   ~flyers.FlyerInterface.describe_collect
   ~flyers.FlyerInterface.collect

Asynchronous status
===================

Hardware control and data collection is an inherently asynchronous
activity.  The many devices on a beamline are (in general) uncoupled
and can move / read independently.  This is reflected in the API as
most of the methods in :obj:`BlueskyInterface` returning `Status`
objects and in the callback registry at the core of
:obj:`~ophydobj.OphydObject`.  The :class:`StatusBase` objects are the
bridge between the asynchronous behavior of the underlying control
system and the asynchronous behavior of
:class:`~bluesky.run_engine.RunEngine`.

The core API of the status objects is a property and a private method:

.. autosummary::
   :nosignatures:

   status.StatusBase.finished_cb
   status.StatusBase._finished

The `bluesky`_ side assigns a callback to
:attr:`status.StatusBase.finished_cb` which is triggered when the
:meth:`status.StatusBase._finished` method is called.  The status object
conveys both that the action it 'done' and if the action was
successful or not.



Callbacks
=========

The base class of almost all objects in ``ophyd`` is :obj:`~ophydobj.OphydObject`
 a callback registry

.. autosummary::
   :toctree: _as_gen
   :nosignatures:

   ~ophydobj.OphydObject
   ~ophydobj.OphydObject.event_types
   ~ophydobj.OphydObject.subscribe
   ~ophydobj.OphydObject.clear_sub

   ~ophydobj.OphydObject._run_subs

   ~ophydobj.OphydObject._reset_sub

This registry is used to connect to the underlying events from the
control system and propagate them up to bluesky, either via
`~status.StatusBase` objects or via direct subscription from the
:class:`~bluesky.run_engine.RunEngine`.
