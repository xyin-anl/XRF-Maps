> [!IMPORTANT]
> This forked version focuses on the Python interface and may be slightly different from the upstream version.

# XRF-Maps

X-ray fluorescence (XRF) imaging typically involves the creation and analysis of 3D data sets, where at each scan position a full energy dispersive x-ray spectrum is recorded. This allows one to later process the data in a variety of different approaches, e.g., by spectral region of interest (ROI) summation with or without background subtraction, principal component analysis, or fitting. `XRFMaps` is a C++ open source software package that implements these functions to provide a tool set for the analysis of XRF data sets. It is based on the original IDL implementation, [`MAPS`](https://www.aps.anl.gov/Microscopy/Software-and-Tools-MAPS). Recently, a differentiable PyTorch-based implementation, [`MapsTorch`](https://github.com/xyin-anl/MapsTorch) has been developed that enables Automatic Differentiation (AD)-based XRF fitting and data analysis.

# Compiling on Linux with Python bindings
```bash
# Need gcc 6.0 or greater and cmake 3.5 or greater
git clone --recurse-submodules https://github.com/xyin-anl/XRF-Maps.git
cd XRF-Maps
mkdir build
cd vcpkg
./bootstrap-vcpkg.sh
./vcpkg install hdf5 netcdf-c yaml-cpp cppzmq nlopt jsoncpp
cd ../build
export PYTHON_BASE_DIR=/home/user/miniconda3/envs/mapstorch # Replace with your python base directory
export PYTHON_VERSION=3.11 # Replace with your python version
cmake -DCMAKE_TOOLCHAIN_FILE=../vcpkg/scripts/buildsystems/vcpkg.cmake -DBUILD_WITH_ZMQ=ON -DBUILD_WITH_PYBIND11=ON -DPYTHON_EXECUTABLE=$PYTHON_BASE_DIR/bin/python -DPYTHON_LIBRARY=$PYTHON_BASE_DIR/lib/libpython3.so -DPYTHON_INCLUDE_DIR=$PYTHON_BASE_DIR/include/python$PYTHON_VERSION ..
make
```
After compilation, you will see the Python module and `xrf_maps` executable in the bin directory. For other compiling options, please check upstream repository.

# Python Interface

The Python interface (`pyxrfmaps`) provides comprehensive bindings to the C++ library, enabling XRF data analysis directly from Python. Here's a detailed overview of the main components and their usage:

## Core Components

### Element Information Management
```python
import pyxrfmaps as px

# Load element information from reference files
px.load_element_info(element_henke_filename, element_csv_filename)

# Access element info
element_info_map = px.ElementInfoMap()
element = element_info_map.get_element("Fe")  # Get by symbol
element = element_info_map.get_element(26)    # Get by atomic number
```

### Data Structures

#### Spectra Classes
- `Spectra`: Single spectrum data structure
- `Spectra_Line`: 1D array of spectra
- `Spectra_Volume`: 2D array of spectra
```python
# Working with spectra
spectra = px.Spectra()
spectra_line = px.Spectra_Line()
spectra_volume = px.Spectra_Volume()

# Get dimensions
rows = spectra_volume.rows()
cols = spectra_volume.cols()
samples = spectra_volume.samples_size()
```

#### Fitting Parameters
```python
# Create and manipulate fit parameters
fit_params = px.Fit_Parameters()
fit_params.add_parameter(px.Fit_Param("param_name", value))
fit_params.update_values(new_values)
```

### Fitting Models and Routines

#### Models
The package provides Gaussian-based models for XRF spectrum fitting:
```python
# Create Gaussian model
model = px.fitting.models.GaussModel()
model.reset_to_default_fit_params()
model.update_fit_params_values(fit_params)
```

#### Fitting Routines
Several fitting routines are available:
```python
# ROI-based fitting
roi_routine = px.fitting.routines.roi()

# Parameter-optimized fitting
param_routine = px.fitting.routines.param_optimized()

# Matrix-based fitting
matrix_routine = px.fitting.routines.matrix()

# Non-negative least squares fitting
nnls_routine = px.fitting.routines.nnls()

# SVD-based fitting
svd_routine = px.fitting.routines.svd()
```

#### Optimizers
```python
# NLOPT optimizer
optimizer = px.fitting.optimizers.nlopt()
fit_routine.set_optimizer(optimizer)
```

### Working with Files

#### Loading and Saving Data
```python
# Load dataset
filename = "dataset.h5"
spectra_volume = px.load_spectra_volume(filename)

# Save results
px.io.file.hdf5_save_element_fits(output_path, element_counts, row_start, row_end, col_start, col_end)
```

### Workflow Components

The package provides workflow components for processing streaming data:
```python
# Create file source for streaming processing
source = px.SpectraFileSource()
source.set_init_fitting_routines(True)

# Create network streamer for distributed processing
streamer = px.SpectraNetStreamer("tcp://localhost:5555")
streamer.set_send_spectra(True)
```

## Complete Example

Here's a complete example showing how to perform XRF fitting on a dataset:

```python
import pyxrfmaps as px

element_csv_filename = "/reference/xrf_library.csv"
element_henke_filename = "/reference/henke.xdr"
detector_num = 0
dataset_dir = '/example_dataset/'
dataset = 'bnp_fly0001.mda.h50'
full_path = dataset_dir + 'img.dat/' + dataset

# initialize element info
px.load_element_info(element_henke_filename, element_csv_filename)

# Load dataset
int_spec = load_dataset(full_path)

# Load fit parameters 
po = px.load_override_params(dataset_dir, detector_num, True)

# Use Gausian-based Model
model = px.fitting.models.GaussModel()

# Select fitting routine and optimizer
fit_rout = px.fitting.routines.param_optimized()
opt = px.fitting.optimizers.nlopt()
fit_rout.set_optimizer(opt)

# Initialize model and fit routine with fit parameters
energy_range = px.get_energy_range(int_spec.size, po.fit_params)
model.update_fit_params_values(po.fit_params)
fit_rout.initialize(model, po.elements_to_fit, energy_range)

# Fit parameters
params = fitting_routine.fit_params(model, int_spec, po.elements_to_fit, False)

# You may need to manually update override parameter files here 

# Fit element counts
counts = fit_rout.fit_counts(model, int_spec, po.elements_to_fit)

# Get Fit Spectra 
fit_spec = fit_rout.fit_spectra(model, int_spec, po.elements_to_fit)

# Resize int_spec to match fit_spec
int_spec = int_spec[energy_range.min:energy_range.max+1]
```

## Additional Features

### Background Subtraction
```python
# Perform SNIP background subtraction
background = px.snip_background(spectra, width, energy_offset, energy_slope, energy_quad, first_width, max_iterations)
```

### Quantification
```python
# Perform quantification analysis
px.perform_quantification(params_override, element_counts, quantification_standards)
```

For more advanced usage, please refer to `src/pybindings/main.cpp` and the upstream repository.