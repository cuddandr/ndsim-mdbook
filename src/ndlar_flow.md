# Running ndlar-flow

This documenation primarily concerns only running the ndlar-flow stage of the ND simulation and not details on how the code works (for now). More details on ndlar-flow can be found in the [repository README](https://github.com/DUNE/ndlar_flow/blob/develop/README.rst).

## Installation

The package is available in the [ndlar-flow repository](https://github.com/DUNE/ndlar_flow) as part of the [DUNE GitHub organization](https://github.com/DUNE) and is installed via Pip. There is one dependency that also needs to be installed manually, which is [h5flow](https://github.com/DUNE/h5flow):
```bash
$> git clone -b main https://github.com/DUNE/h5flow.git
$> cd h5flow
$> pip install .
```
Then ndlar-flow can be installed in a similar manner with pip:
```bash
$> git clone https://github.com/DUNE/ndlar_flow.git
$> cd ndlar_flow
$> pip install .
```

It is highly recommended to use a virtual environment to isolate the ndlar-flow environment. To create and activate a virtual environment, run:
```bash
$> python3 -m venv flow-env
$> source flow-env/bin/activate
```
where `flow-env` is the name of the directory where the virtual environment will be stored (and can be changed).

## Running ndlar-flow

Running ndlar-flow is actually performed with the `h5flow` executable from the h5flow package where the various steps are defined using YAML configuration files. Each step or module for ndlar-flow has a YAML spec found in [ndlar-flow/yamls](https://github.com/DUNE/ndlar_flow/tree/develop/yamls) that run code defined in [ndlar-flow/src](https://github.com/DUNE/ndlar_flow/tree/develop/src).

To run a single module of ndlar-flow:
```bash
$> h5flow -c /path/to/spec.yaml -i /path/to/input.hdf5 -o /path/to/output.hdf5
```
Multiple modules can be run on a single file by providing a list of YAML specs to the `-c` flag:
```bash
$> h5flow -c /path/to/step1.yaml /path/to/step2.yaml /path/to/step3/yaml -i /path/to/input.hdf5 -o /path/to/output.hdf5
```

The "standard" workflow for 2x2, FSD, and ND-LAr consists of the following steps with different versions specific to each detector *and* different versions between data and MC processing:
+ charge event building
+ charge event reconstruction
+ combined reconstruction
+ (charge) prompt calibration
+ (charge) final calibration (this is not always run)
+ light event building
+ light event reconstruction
+ charge/light association

Each step can be started one at a time, or all can be queued to run together. For example, to run all the above steps using three separate invocations of `h5flow` for **2x2 MC** files:
```bash
# charge workflows
workflow1='yamls/proto_nd_flow/workflows/charge/charge_event_building_mc.yaml'
workflow2='yamls/proto_nd_flow/workflows/charge/charge_event_reconstruction_mc.yaml'
workflow3='yamls/proto_nd_flow/workflows/combined/combined_reconstruction_mc.yaml'
workflow4='yamls/proto_nd_flow/workflows/charge/prompt_calibration_mc.yaml'
workflow5='yamls/proto_nd_flow/workflows/charge/final_calibration_mc.yaml'

# light workflows
workflow6='yamls/proto_nd_flow/workflows/light/light_event_building_mc.yaml'
workflow7='yamls/proto_nd_flow/workflows/light/light_event_reconstruction_mc.yaml'

# charge-light trigger matching
workflow8='yamls/proto_nd_flow/workflows/charge/charge_light_assoc_mc.yaml'

# Optional, but recommended, compression of the output files
compression='-z lzf'

# Ensure that the second h5flow doesn't run if the first one crashes.
set -o errexit

h5flow -c $workflow1 $workflow2 $workflow3 $workflow4 $workflow5 \
    -i "$inFile" -o "$outFile" $compression

h5flow -c $workflow6 $workflow7 \
    -i "$inFile" -o "$outFile" $compression

h5flow -c $workflow8 \
    -i "$outFile" -o "$outFile" $compression
```
Note that the YAML specs have identifiers and paths for `proto_nd_flow` (the 2x2) and for MC processing. To run for FSD, ND-LAr, or data processing, change the YAML specs to the corresponding versions.

## ND_Production

The [ND_Production repository](https://github.com/DUNE/ND_Production) has scripts for installing and running ndlar-flow located in `run-ndlar-flow/` where the scripts assume to be working within the production environment.

Install ndlar-flow by running `run-ndlar-flow/install_ndlar_flow.sh` and it will create a virtual environment and install the most recent commit on the `develop` branch of ndlar-flow and its necessary dependencies, including `h5flow`. Once ndlar-flow is installed, the environment can be activated by `source flow.venv/bin/activate`.

Run ndlar-flow by running `run-ndlar-flow/run_ndlar_flow.sh` (pay attention to the dashes vs underscores), but it assumes the production environment is properly set through a series of shell environment variables starting with `ND_PRODUCTION_`.
