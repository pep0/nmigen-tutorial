# Simulating

The best way to simulate a module is through nMigen's `Simulator`.

## Define your ports

Define a `ports` function in your module which returns an array of your module's ports:

```python
class YourModule(Elaboratable):
    ...
    def ports(self):
        return [self.yourmodule.p1, self.yourmodule.p2, ...]
```

## Create a top-level module

Create a top-level module for your simulation:

```python
from nmigen import *
from nmigen.back.pysim import Simulator, Delay, Settle
from somewhere import YourModule

if __name__ == "__main__":
    m = Module()
    m.submodules.yourmodule = yourmodule = YourModule()

    sim = Simulator(m)

    def process():
        # To be defined

    sim.add_process(process) # or sim.add_sync_process(process), see below
    with sim.write_vcd("test.vcd", "test.gtkw", traces=yourmodule.ports()):
        sim.run()
```

> There is currently a bug in nMigen where inputs to your module are not output to the trace file. To get around this, for each such input, place this in your `main` **before** the `Simulator` construction:
> ```python
>     input1 = Signal(...)
>     m.d.comb += yourmodule.input1.eq(input1)
>     ...
>     sim = Simulator(m)
> ```
> Inside your `process`, refer to this input as `input1`, not `yourmodule.input1`. This will force nMigen to include `input1` in the trace file.

## Define your clocks, if any

If you have clocks, add each clock after the `Simulator` construction, giving the clock period in seconds. For example, a 1MHz clock for clock domain `fast_clock` and a 1.1MHz clock for `faster_clock` would be:

```python
   sim = Simulator(m)
   sim.add_clock(1e-6, domain="fast_clock")
   sim.add_clock(1.1e-6, domain="faster_clock")
```

Leaving out `domain` will cause the clock period to be assigned to the default clock domain, `sync`.

## The process function

The `process` function is a Python generator that nMigen calls to see what to do next in the simulation. Since it is a generator, `process` must `yield` a statement to perform. For example:

```python
    def process():
        yield x.eq(0)
        yield y.eq(0xFF)
```

The above would set `x` to 0 and `y` to 0xFF, with effectively no delay between them.

You can yield nMigen `Value`s, which you can then use, for example, to do various comparisons:

```python
    def process():
        yield x.eq(0)
        yield Delay(1e-6)  # causes a delay of 1 microsecond
        yield y.eq(0xFF)
        yield Settle()     # forces all combinatorial computation to happen
        got = yield yourmodule.sum
        want = yield (x+y)[:8]
        if got != want:
            print(f"Oh noes! Error! Got {got:02x}, wanted {want:02x}")
```

In the above example, `x` will be set to 0, then there will be a one microsecond delay, then `y` will be set to 0xFF, all combinatorial logic will be given a chance to settle, and finally `yourmodule.sum` and `(x+y)[:8]` will be evaluated, and if they are not equal, a diagnostic message is sent to the terminal output.

You can even have more than one process running at a time!

```python
    def x_process():
        yield Delay(1e-6)
        yield x.eq(0)
        yield Settle()

    def y_process():
        yield Delay(1.1e-6)
        yield y.eq(0xFF)
        yield Settle()

    sim.add_process(x_process)
    sim.add_process(y_process)
```

In the above example, `x` will be set to 0 at time 1 microsecond, and `y` will be set to 0xFF at time 1.1 microseconds.

**Warning**: driving the same signal from more than one process can lead to undefined behavior if both processes assign to the signal simultaneously.

### Non-clocked processes

If your module doesn't use a clock, then you can create a non-clocking `process`. You must add such a process to the simulator via `add_process`:

```python
    sim.add_process(process)
```

Then you must manipulate all the input signals for yourself.

### Clocked processes

If your module has one or more clocks, then you can create a clocking `process`. For each clock domain, you must add such a process to the simulator via `add_sync_process`:

```python
    sim.add_sync_process(process1, domain="name1")
    sim.add_sync_process(process2, domain="name2")
```

If you `yield` with no value from a clocking process, then one cycle of the clock will occur. Note that for clocked processes, one clock edge will always occur before the process starts, so take that into account when you look at your traces.

It is also important to understand when statements are executed in relation to clock edges. They are always executed **infinitesimally after** the previous clock edge. Thus, in this example:

```python
    def process():
        yield x.eq(0)  # step 1
        yield          # step 2
        yield x.eq(1)  # step 3
        yield          # step 4
```

there will be one clock edge that always takes place before the process runs. Then `x` is set to 0 (step 1). Then another clock edge happens (step 2). `x` is set to 1 infinitesimally after that clock edge (step 3). Then another clock edge happens (step 4). In the traces, you will not see signals change "just after" the clock edge. They will appear to be coincident with the clock edge. Just remember the rule that signals that appear to change coincident with a clock edge actually change just after that clock edge. Outputs that change like that can be considered to have been "caused" by the clock edge.

### Passive and active processes

This section is delayed until I can understand what these are.

## Running the simulation and viewing the output

The simulation is run simply by running the main module:

```
$ python3 main_module.py
```

The output should be a `test.vcd` file and a `test.gtkw` file. Running `gtkwave` will allow you to view the output. Running it on `test.vcd` will make you select the signals you want to see when `gtkwave` opens, while running it on `test.gtkw` will open `gtkwave` showing the signals in the `traces` key that you gave in the call to `sim.write_vcd()`.

```
$ gtkwave test.vcd
$ gtkwave test.gtkw
```
