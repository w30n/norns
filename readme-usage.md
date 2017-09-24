# norns usage

---------------

## launching components

### 1. launch `crone` (audio engine)

run `crone.sh` from the norns directory. this creates a `sclang` process wrapped with `ipc-wrapper`, and a pair of ipc sockets in `/tmp`.

if the crone classes are installed correctly, you should see some lines like this in output from sclang initialization: 
```
-------------------------------------------------
 Crone startup

 OSC rx port: 57120
 OSC tx port: 8888
--------------------------------------------------
```

and immediately after sclang init, you should see the server being booted and some jack/alsa related messages. 

### 2. launch `matron` (lua interpreter)

with the audio engine running, run `matron.sh` from the norns directory. this creates a `matron` process wrapped with `ipc-wrapper`, and a pair of ipc sockets in `/tmp`.

### 3. launch `maiden` (UI client)

with the other components running, run the `maiden` executable from anywhere. this presents the user with a REPL interface to both the lua interpreter and the sclang backend.

special characters:

- `TAB` : switch between lua and sclang REPLs
- `shift+TAB` : switch between output log and input log for each REPL
(_**TODO: input log isn't really implemented**_)
- `q` : quits the client.

_**TODO: output scrolling works, but no commands for it yet.**_

_**TODO: more robust command input, e.g. `ctl-a` as an escape sequence.**_

---------------

## lua programming

_**FIXME: library structure could be more elegant/idiomatic (no underscores.)**_

### startup

the file `scripts/startup.lua` is executed on startup. execution takes place after the audio engine is booted, but before input devices are scanned, so device connection callbacks can be here, and `load_engine()` is available.

### audio

audio processing is performed by audio *engines*. only one engine is loaded at a time. each engine presents an arbitrary collection of 'commands'. each command consists of a name and zero or more arguments. each argument can be one of three types: `int`, `float`, or `string`. thus, engine commands map directly to a subset of OSC messages.

#### engine control functions:
- `report_engines()` : request a list of available engines

- `load_engine(name)` : request audio server to load the named engine


- `report_commands()` : request the current audio engine to list available commands, and populate the `engine` table with command functions (see below)


- `send_command(idx, ...)` : send an indexed command with a variable number of arguments. 


additionally, specific engine command functions are created dynamically on reception of a command list from the audio server, and placed in the table `norns.engine`. so for example, the `TestSine` engine reports just two commands, each of which takes a single float argument:
```
1: hz (f)
2: amp (f)
```

on receiving this report, norns creates two functions whose definitions would look like this:
```
norns.engine.hz = function(arg1) 
  send_command(1, arg1)
end
norns.engine.amp = function(arg1) 
  send_command(2, arg1)
end

```

it is recommended to set a shortcut on startup: `e = norns.engine`; the user then can then simply use `e.hz(440)` and `e.amp(0.5)` in this example.

#### engine callbacks:

- `report.engines(names, count)` : called when an engine report is ready. arguments: table of engine names (strings), number of engines.

- `report.commands(commands, count)` : called when a command report is ready. the `commands` argument is a table of tables; each subtable is of the form `{ name, format }`, where `name` is the name of the command and `format` is an OSC-style format string. 

note that commands are reported automatically on engine load. so for the time being, the `report.commands` callback is the easiest method for delaying lua code execution until an engine is finished loading.


### I/O devices

#### monome 

grid and arc devices can be hotplugged. connected devices are available as tables in `norns.grid.devices` and `norns.arc.devices`. each table includes information about the device as well as methods to control its output.

- `Grid:led(x, y, z)` : set a single led at `(x,y)` to brightness `z`, in the range 0-15.

- `Grid:refresh()` : update the device's physical state.

_**TODO: arc**_

for device hotplug and gesture input, the following callbacks can be defined:

- `grid.add(device)` : grid device was added. the argument is a `Grid` table, which includes the following fields:
    - `id` : an integer ID. these never repeat during a given run of `matron`.
	- `serial` : a serial string representing the device, like `m1000404`.
	- `name` : a human-readable string describing the device, like `monome 128`.
	- `dev` : an opaque pointer to the raw device handle. this is passed back to C on device update; user scripts shouldn't need to use it.

- `grid.remove(id)` : grid device was removed

- `grid.key(device, x, y, value)` : key event was received on the given device. 

_**TODO: arc**_

#### HID

HID input devices work similarly to monome devices. however, the event structure is necessarily more complex. 

callbacks:

- `input.add(device) ` : an input device was added. argument is an `InputDevice` table, with the following fields:
    - `id` : an integer ID. these never repeat during a given run of `matron`.
	- `serial` : a serial string representing the device. for now, this is an 8-digit hex string; the first 4 hex digits are the product ID, the last 4 are the vendor ID.
	- `name` : a human-readable string desribing the device (e.g. "Logitech USB Optical Mouse.")
    - `types` : event types supported by this device. 
	- `codes` : a table containing one subtable per supported event type, indexed by event type. each subtable contains one entry per supported event code for that event type; index in subtable is code number, value is code name.
	
	event type and code names are defined in `sys/input_device_codes.lua`. 

- `input.remove()` : _**TODO: not connected yet**_

- `input.event(device, type, code, value)` : respond to an event from the given device.

_**TODO?: HID output**_

#### MIDI

_**TODO**_

### timers

`matron`  maintains a fixed number of high-resolution timers that can be used from lua:

- `timer(index, stage)` : this shared callback function is called whenever any timer fires. arguments are the timer's index and current stage number. *timer index and stage number are 1-based.*

- `timer_start(index, period, count, stage)` : start a timer. the first callback happens immediately. if the timer  is already running, it will be restarted from the given stage.
    - index: 1-based index of the timer.
	- period: seconds between stages. if ommitted, the previous setting for this timer is re-used.
	- count: number of callbacks to perform before stopping. if ommitted, nil, or <=0, the timer will run indefinitely.
	- stage: stage number to start at (1-based.) default is 1.
	
- `timer_stop(index)` : stop the indexed timer immediately. 
		
### graphics

_**TODO**_