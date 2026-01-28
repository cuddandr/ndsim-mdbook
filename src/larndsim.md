# Running larnd-sim

## Installation

The package is available on the DUNE GitHub repository and is installed via Pip:
```bash
$> git clone https://github.com/DUNE/larnd-sim.git
$> cd larnd-sim
$> pip install .
```
The simulation requires GPUs and the installation will attempt to install the appropriate version of `cupy` for your system. Larnd-sim has a couple optional set of dependencies aimed at development that add functionality for linting, type-checking, and wrapper for running larnd-sim.

It is highly recommended to use a virtual environment to isolate the larnd-sim environment. To create and activate a virtual environment, run:
```bash
$> python3 -m venv larndsim-env
$> source larndsim-env/bin/activate
```
where `larndsim-env` is the name of the directory where the virtual environment will be stored (and can be changed).

## Running larnd-sim

Larnd-sim has two main options to run the code: `simulate_pixels.py` and `lar_runner.py`. The `simulate_pixels.py` file is the direct code to run larnd-sim, and `lar_runner.py` is a wrapper that builds the command with arguments for `simulate_pixels.py`.

Reminder that running larnd-sim requires a GPU. Either ensure one is available on the machine or request one from a computing cluster. If running at NERSC, see the chapter on [NERSC job submission](nersc.md) to request a GPU.

### simulate_pixels.py

The simplest method to call `simulate_pixels.py` is to specify a detector configuration, input file, and output file. For example:
```bash
simulate_pixels.py --config CONFIG_KEYWORD --input_filename INPUT_FILENAME --output_filename OUTPUT_FILENAME
```
The available configurations for `CONFIG_KEYWORD` can be found in [larndsim/config/config.yaml](https://github.com/DUNE/larnd-sim/blob/develop/larndsim/config/config.yaml) as the names of the YAML tables. The most common configurations are `2x2`, `ndlar`, and `fsd`.

The command line interface for `simulate_pixels.py` also supports specifying the configuration via flags, for example:
```bash
$> simulate_pixels.py \
--input_filename=INPUT_FOR_A_2x2_BEAM_EXAMPLE.h5 \
--output_filename=OUTPUT_FOR_A_2x2_BEAM_EXAMPLE.h5 \
--mod2mod_variation=False \
--pixel_layout=larndsim/pixel_layouts/multi_tile_layout-2.4.16.yaml\
--detector_properties=larndsim/detector_properties/2x2.yaml \
--response_file=larndsim/bin/response_44.npy \
--light_simulated=True \
--light_lut_filename=larndsim/bin/lightLUT.npz \
--light_det_noise_filename=larndsim/bin/light_noise_2x2_4mod_July2023.npy
```
Note that the default configuration is always active, so in order to control the configuration each relevant option needs to be specified. However this can be used to then override only specific parts of the default configuration.

Running `simulate_pixels.py --help` will display a man-page for the options and their descriptions.

### lar_runner.py

To use `lar_runner.py`, larnd-sim needs to be installed using `pip install .[runner]` to grab both the required dependencies and the optional packages to use `lar_runner.py`. This wrapper script was written to handle building the complex comands for profiling using NSight Systems, and can run larnd-sim without the profiling tools. It uses a YAML configuration and optional command line arguments to configure the simulation and profiling options.

There is an example in [larnd-sim/cli/runner.yaml](https://github.com/DUNE/larnd-sim/blob/develop/cli/runner.yaml) for using `lar_runner.py` and should be edited as necessary. Most options can also be specified on the command line, which may be easier for processing multiple files.

The general design is to use a common YAML configuration file with a few global options and sub-commands for each of the different capabilities that include their own options. `lar_runner.py` and each of the sub-commands have `-h, --help` flags that will list the available options and a short description. Specifying an option on the command line will override the same option inside the YAML file; command-line arguments take precedence. If unspecified, `lar_runner.py` attempts to read the configuration in `larnd-sim/cli/runner.yaml`.

To run larnd-sim using the settings inside the YAML:
```bash
$> lar_runner.py -y /path/to/config.yaml run
```
and will use default variables where defined.

To specify the input, output, etc. on the command line and use the YAML for everything else:
```bash
$> lar_runner.py -y /path/to/config.yaml run -c 2x2 -i /path/to/input_edepsim.hdf5 -o /path/to/output_larndsim.hdf5
```

Additionally you can pass `-d` to perform a "dry run" which will print out what the command will do, but not actually run, if you want to check will be done. For example:
```bash
$> lar_runner.py -y /path/to/config.yaml -d run
```

The entire `--help` message documentation:
```bash
usage: lar_runner.py [-h] [-v] [-y YAML] [-d] [-f] COMMAND ...

Command line tool for running, profiling, and comparing larnd-sim

positional arguments:
  COMMAND               Available commands
    run                 Run larnd-sim
    nsys                Run Nsight Systems profiling
    ncu                 Run Nsight Compute profiling
    compare             Compare larnd-sim files
    report              Generate nsys profile summary

options:
  -h, --help            show this help message and exit
  -v, --verbose         Enable verbose output
  -y YAML, --yaml YAML  YAML file containing common configuration options.
  -d, --dry_run         Print command but do not execute
  -f, --force           Overwrite existing output files.
```

#### run (larnd-sim)

The `run` sub-command is for running larnd-sim without any profiling, and reads options from the `larnd-sim` YAML table in the configuration file. Note that not all options from `simulate_pixels.py` are available in the YAML config; it is largely intended to use a predefined configuration, but any flag that is valid for `simulation_pixels.py` can be passed via the `--args` flag.

The available YAML options:
```yaml
larnd-sim:
  config: "2x2" # CONFIG KEYWORD for simulate_pixels.py
  input_file: "/path/to/input_edepsim.hdf5"
  output_file: "/path/to/output_larndsim.hdf5"
  rng_seed: 321 # RNG seed for simulation
  compression: 'lzf' # Optional compression, same as simulate_pixels.py
  n_events: 75 # Optional limit on the numebr of events to process, same as simulate_pixels.py; leave unspecified to process all events
```

## ND_Production

The ND_Production repository has scripts for installing and running larnd-sim with the production environment located in `run-larnd-sim/`.
