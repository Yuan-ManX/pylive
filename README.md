# PyLive

**NOTE: Work is currently underway on updating pylive to interface with [AbletonOSC](https://github.com/ideoforms/AbletonOSC) for Live 11 support, hopefully for completion in December 2022.**

PyLive is a framework for querying and controlling Ableton Live from a standalone Python script, mediated via Open Sound Control. It is effectively an interface to the Live Control Surfaces paradigm, which means it can do anything that a hardware control surface can do, including:

 - query and modify global parameters such as tempo, volume, pan, quantize, arrangement time
 - query and modify properties of tracks, clips, scenes and devices
 - trigger and stop clips and scenes

It can perform most of the operations described in the [LiveOSC OSC API](https://github.com/hanshuebner/LiveOSC/blob/master/OSCAPI.txt).

If you are looking simply to send MIDI messages to Live, this module is not what you want. Instead, try setting up a [virtual MIDI bus](https://help.ableton.com/hc/en-us/articles/209774225-How-to-setup-a-virtual-MIDI-bus) and using [isobar](https://ideoforms.github.io/isobar/) to generate MIDI sequences.

## Requirements

* [Ableton Live 9+](http://www.ableton.com/live)
* [Python 2.6+](http://www.python.org)
* [LiveOSC (fork)](https://github.com/ideoforms/LiveOSC): A maintained fork of the [LiveOSC](http://livecontrol.q3f.org/ableton-liveapi/liveosc/) MIDI control script, updated to work with Live 9.6 and 10. Must be installed in Live's `MIDI Remote Scripts` (see [README](https://github.com/ideoforms/LiveOSC))
* [liblo](http://liblo.sourceforge.net/): Install via [Homebrew](https://brew.sh/) with `brew install liblo`. See "Alternate installation without liblo" below if you can't install liblo.

## Installation

From PyPi:

```
pip3 install Cython # needed for pyliblo
pip3 install pylive
```

Or via git:
```
git clone https://github.com/ideoforms/pylive.git
cd pylive
python3 setup.py install
```

To check that pylive is communicating successfully with Ableton Live, try running one of the [examples](examples), or run the test suite with:
```
python3 setup.py test
```

### Alternate installation without liblo

In some environments (particularly Windows), liblo can be a pain to get running. In these scenarios, an alternative backend is provided that uses the `pythonosc` backend for OSC communications.

If you want to install without liblo support:

* clone the repo from GitHub
* remove the `install_requires` line from `setup.py`
* install pylive with `python3 setup.py install`

You will then need to set an environmental variable specifying the `pythonosc` backend before running scripts using pylive:

```
export PYLIVE_BACKEND='pythonosc'
python3 your-script.py
```

## Usage

```python
#------------------------------------------------------------------------
# Basic example of pylive usage: scan a Live set, trigger a clip,
# and modulate some device parameters.
#------------------------------------------------------------------------
import live
import random

#------------------------------------------------------------------------
# Scan the set's contents and set its tempo to 110bpm.
#------------------------------------------------------------------------
set = live.Set()
set.scan(scan_clip_names = True, scan_devices = True)
set.tempo = 110.0

#------------------------------------------------------------------------
# Each Set contains a list of Track objects.
#------------------------------------------------------------------------
track = set.tracks[0]
print("Track name %s" % track.name)

#------------------------------------------------------------------------
# Each Track contains a list of Clip objects.
#------------------------------------------------------------------------
clip = track.clips[0]
print("Clip name %s, length %d beats" % (clip.name, clip.length))
clip.play()

#------------------------------------------------------------------------
# We can determine our internal timing based on Live's timeline using
# Set.wait_for_next_beat(), and trigger clips accordingly.
#------------------------------------------------------------------------
set.wait_for_next_beat()
clip.get_next_clip().play()

#------------------------------------------------------------------------
# Now let's modulate the parameters of a Device object.
#------------------------------------------------------------------------
device = track.devices[0]
parameter = random.choice(device.parameters)
parameter.value = random.uniform(parameter.minimum, parameter.maximum)
```

## Overview

To begin interacting with an Ableton Live set, the typical workflow is as follows. Live should normally be running on localhost, with LiveOSC enabled as a Control Surface.

* Create a `live.Set` object.
* Call `set.scan()`, which queries Live for an index of tracks, clip statuses, and (optionally) clip names and devices
* Interact with Live by setting and getting properties on your `Set`:
  * `set.tempo`, `set.time`, `set.overdub` are global Set properties
  * `set.tracks` is a list of Track objects
  * `set.tracks[N].name`, `set.tracks[N].mute`, are Track properties
  * `set.tracks[N].clips` is a list of Clip objects (with empty slots containing `None`)
  * `set.tracks[N].devices` is a list of Device objects
  * `set.tracks[N].devices[M].parameters` is a list of Parameter objects

Getters and setters use Python's `@property` idiom, meaning that accessing `set.tempo` will query or update your Live set.

If you know that no other processes will interact with Live, set `set.caching = True` to cache properties such as tempo. This will query the Live set on the first instance, and subsequently return locally-stored values.

For further help, see `pydoc live`.

## Classes

* `Set`: Represents a single Ableton Live set in its entirety. 
* `Track`: A single Live track object. Contains `Device` and `Clip` objects. May be a member of a `Group`.
* `Group`: A grouped set of one or more `Track` objects.
* `Device`: An instrument or audio effect residing within a `Track`. Contains a number of `Parameter` objects.
* `Parameter`: An individual control parameter of a `Device`, with a fixed range and variable value.

## Limitations

Note that pylive is not intended for sending MIDI note events or control messages to a set. For MIDI controls, use a separate module such as [mido](https://mido.readthedocs.io).
