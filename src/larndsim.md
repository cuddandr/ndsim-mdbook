# Running larnd-sim

## lar_runner.py

Checkout a copy of larnd-sim as normal and install using `pip install .[runner]` to grab both the required dependencies and the optional packages to use `lar_runner.py`

`lar_runner.py` is a wrapper script to run larnd-sim with and without profiling and uses a YAML configuration file.

There is an example in [larnd-sim/cli/runner.yaml](https://github.com/DUNE/larnd-sim/blob/develop/cli/runner.yaml) and this should be edited for your needs. Most options can also be specified on the command line, which may be easier for processing multiple files.

To run larnd-sim using the settings inside the YAML:
```bash
$> lar_runner.py -y /path/to/config.yaml run
```

To specify the input, output, etc. on the command line:
```bash
$> lar_runner.py -y /path/to/config.yaml run -c 2x2 -i /path/to/input_edepsim.hdf5 -o output_larndsim.hdf5
```
Running with the `-h` or `--help` will display the options.

Additionally you can pass `-d` to perform a "dry run" which will print out what the command will do, but not actually run, if you want to check will be done. For example:
```bash
$> lar_runner.py -y /path/to/config.yaml -d run
```
