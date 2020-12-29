Source location: https://github.com/openvinotoolkit/open_model_zoo/tree/master/tools/downloader/README.md


# Model Downloader and Automation Tools

This directory contains scripts that automate model-related tasks
based on configuration files in the models' directories. Use these scripts 
instead of trying to parse the configuration files. The configuration file 
formats are undocumented and might change.

These scripts are:

* `downloader.py` (model downloader) downloads model files from online sources
  and, if necessary, modifies them to increase usability with the Model
  Optimizer;

* `converter.py` (model converter) uses the Mode Optimizer to convert models that 
  are not in the Inference Engine IR format to the correct format.

* `quantizer.py` (model quantizer) uses the using Post-Training Optimization Toolkit 
  to quantize full-precision models in the IR format into low-precision versions.

* `info_dumper.py` (model information dumper) prints information about the models
  in a stable machine-readable format.


## Preparation

1. Install Python* version 3.6 or higher.
2. Install the tool dependencies:
	```sh
	python3 -mpip install --user -r ./requirements.in
	```

3. For the model converter tool: Install the OpenVINO&trade; toolkit and the prerequisite 
libraries for Model Optimizer. See the [OpenVINO toolkit documentation](https://docs.openvinotoolkit.org/) 
for details.

4. Install framework dependencies  :

- For Caffe2 models:
	```sh
	python3 -mpip install --user -r ./requirements-caffe2.in
	```

- For PyTorch models:
	```sh
	python3 -mpip install --user -r ./requirements-pytorch.in
	```

- For TensorFlow models:
	```sh
	python3 -mpip install --user -r ./requirements-tensorflow.in
	```

## How to use the Model Downloader

To download all models:

```sh
./downloader.py --all
```

Replace `--all` with other filter options to limit the download to a subset of models. 
See the <a href="#shared-options">options applicable to all tools</a>.

| Desired Result | Use Parameter |  Example Command  |
|---|---|---|
| Select a download directory.<br><br><b>Default</b> - Download to a directory tree under the current directory. | `-o`<br>`--output_dir`  | `./downloader.py --all`<br>`--output_dir <DOWNLOAD-DIR>` |
| Specify comma-separated weight precisions to download.  | `--precisions`  | `./downloader.py`<br>`--name face-detection-retail-0004`<br>`--precisions FP16,FP16-INT8`  | 
| Attempt the download more than once.<br><br><b>Default</b> - Try once to download each file. | `--num_attempts`  | `./downloader.py --all` <br>`--num_attempts 5`<br><br>Attempts each download five times.  | 
| Specify a cache directory.<br><br>The script puts a copy of each downloaded file in a cache. If it is already in the cache, <br>get the file from the cache. | `--cache_dir` | `./downloader.py --all`<br>`--cache_dir <CACHE--DIR>` |
| Output results programmatically.<br><br><b>Default</b> - Show the progress as unstructured, human-readable text. | `--progress_format=<option>`<br><br>Options:<br> - `text`<br> - `json` - The output is replaced by a machine-<br>readable progress report. Errors and warnings <br>are in human-readable <br>format.<br><br>See the <a href="#prog-report">JSON progress report format</a>.| `./downloader.py --all`<br>`--progress_format=json` |
| Download files for multiple models at the same time. | `-j`<br>`--jobs` | `./downloader.py --all`<br>`-j8`<br><br>Downloads the maximum of eight models. |


### JSON progress report format<a title="prog-report"> </a>

This section describes the format of the progress report produced by the script when the `--progress_format=json` option is specified.

The report consists of a sequence of events where each event is represented by a line containing a JSON-encoded object. Each event has a member with the name `$type` whose value determines the type of the event and which additional members it contains.

<b>Member Defitions</b>

| Member | Definition | 
|---|---|
| `model` (string) | Model name |
| `model_file` (string) | A file used in the model |
| `num_files` (integer) | Number of files |
| `size` (integer) | File size in bytes |
| `successful` (boolean | Indicates download success |


<br>
<b>Defined Event Types</b>

| Event Type | Members | How it Works |
|---|---|---|
| `model_download_begin` | - `model`<br>- `num_files` | The script started downloading the `model` and will download <br>a total of `num_files` for this model. This event is followed by a <br>corresponding `model_download_end` event. |
| `model_download_end` | - `model`<br> - `successful` | The script stopped downloading `model`.<br>`successful` is true if every file downloaded successfully.  |
| `model_file_download_begin` | - `model`<br> - `model_file`<br> - `size` | The script started downloading the `model_file` for the `model`. <br>`model_file` is `size`.<br><br>`model_file_download_begin` occurs between the `model_download_begin` <br>and `model_download_end` events, and is followed by a <br>corresponding `model_file_download_end` event. |
| `model_file_download_end` | - `model`<br> - `model_file`<br> - `successful` | The script stopped downloading `model_file` for the `model`. <br>`successful` is true if the file was downloaded successfully. |
| `model_file_download_`<br>`progress` | - `model`<br> - `model_file`<br> - `size` | So far the script downloaded `size` of the `model_file` for <br>the `model`. <br>`size` can decrease in a subsequent event if the download <br>is interrupted and retried.<br><br>This event occurs between `model_file_download_begin` <br>and `model_file_download_end` events. |
| `model_postprocessing_begin` | - `model` | The script started post-download processing on the `model`.<br>This event is followed by `model_postprocessing_end` event. |
| `model_postprocessing_end` | - `model` | The script stopped post-download processing on the `model`. |


Tools parsing the machine-readable format should avoid relying on undocumented details.
In particular:

* Tools should not assume that any given event will occur for a given model/file
  (unless specified otherwise above) or will only occur once.

* Tools should not assume that events will occur in a certain order beyond the ordering constraints specified above. In particular, when the `--jobs` option is set to a value greater than 1, event sequences for different files or models may get interleaved.

## How to use the Model Converter

The basic Model Converter command:

```sh
./converter.py --all
```

This command converts all models into the Intermediate Representation format that is required by the Inference Engine. Models that are already in the Intermediate Representation format are ignored. Models in PyTorch and Caffe2 formats are converted to ONNX before they are converted to the Intermediate Representation format.

Replace `--all` with other filter options if you want to convert a subset of models. See the <a href="#shared-options">options applicable to all tools</a>.

| Desired Result | Use Parameter |  Example Command  |
|---|---|---|
| Specify a download tree path.<br><br>The current directory must be the root of a download <br>tree created by the model downloader. | `-d`<br>`--download_dir` | `./converter.py --all` <br>`--download_dir <DOWNLOAD-DIR>` |
| Specify an output directory tree. Models in the <br>Intermediate Representation format are also <br>put in this directory.<br><br><b>Default</b> - the current tree.  | `-o`<br>`--output_dir`  | `./converter.py --all` <br>`--output_dir <OUTPUT-DIR>`  |
| Only produce models in a specific precision. If the <br>specified precision is not supported for a model, <br>that model is skipped.<br><b>Default</b> - Download models in every precision supported <br>for conversion.  | `--precisions` | `./converter.py --all` <br>`--precisions=FP16` |
| Change the environment variables<br><br><b>Default</b> - The Model Optimizer environment variables set <br>by the OpenVINO&trade; toolkit's `setupvars.sh` script in `setupvars.bat`. | `--mo` | `./converter.py --all` <br>`--mo <OPENVINO-DIR>/model_optimizer/mo.py` |
| Add Model Optimizer arguments to those in the model configuration<br><br>You can use this more than once in a command. | `--add_mo_arg` | `./converter.py`<br>`--name=caffenet` <br>`--add_mo_arg=--reverse_input_channels` <br>`--add_mo_arg=--silent` |
| Select a Python executable to use.<br><br><b>Default</b> - Run Model Optimizer with the Python <br>executable that runs the script. | `-p`<br>`--python` | `./converter.py --all`<br>`--python <PYTHON-DIR>` |
| Run up to eight conversion commands at the same time.<br>The option argument must be either:<br> - The maximum number of concurrently executed commands<br> - `auto` - the number of CPUs to use.<br><br><b>Default</b> - Run commands sequentially. | `-j`<br>`--jobs` | `./converter.py --all -j8` <br><br>Run up to eight commands at a time. |
| Print the conversion commands without running them. | `--dry_run` | `./converter.py --all --dry_run` |


Also see the <a href="#shared-options">options applicable to all tools in this document</a>.

## How to use the Model Quantizer

Before you run the model quantizer, you must prepare a directory with the datasets required for the quantization process. This directory is referred to as `<DATASET_DIR>` below. See the <a href="https://github.com/openvinotoolkit/open_model_zoo/blob/develop/datasets.md">Dataset Preparation Guide</a>.

The basic Model Quantizer command:

```sh
`./quantizer.py --all`<br>`--dataset_dir <DATASET_DIR>`
```

This command quantizes all models for which quantization is supported. Other models are ignored.

Replace `--all` with other filter options if you want to quantize a subset of models. 

| Desired Result | Use Parameter |  Example Command  |
|---|---|---|
| Specify a model tree path. The current directory must be the root of the model files<br>created by the model coverter.<br><br> | `-d`<br>`--model_dir` | `./quantizer.py --all`<br>`--dataset_dir <DATASET_DIR>`<br>`--model_dir <MODEL-DIR` |
| Specify an output directory tree. Both the output files and Intermediate Representation files are put in this directory.<br><br><b>Default</b> - The current tree.<br> | `-o`<br>`--output_dir`  | `./quantizer.py --all`<br>`--dataset_dir <DATASET_DIR>`<br>`--output_dir <OUTPUT-DIR>`  |
| Produce models in a specific precision.<br><br><b>Default</b> - models in every supported precision as a quantization output. | `--precisions` | `./quantizer.py --all`<br>`--dataset_dir <DATASET_DIR>` <br>`--precisions=FP16-INT8` |
| Specify the location of the Post-Training Optimization Toolkit.<br><br><b>Default</b> - The environment values set by the OpenVINO&trade; toolkit's `setupvars.sh` or `setupvars.bat` are used. | `--pot` | `./quantizer.py --all`<br>`--dataset_dir <DATASET_DIR>`<br>`--pot <OPENVINO-DIR>/post_training_optimization_toolkit/main.py` |
| Select a Python* executable.<br><br><b>Default</b> - Run Post-Training Optimization Toolkit with the Python executable that was used to run the script. | `-p`<br>`--python` | `./quantizer.py --all`<br>`--dataset_dir <DATASET_DIR>`<br>`--python <PYTHON-DIR>` |
| Specify a target device for Post-Training Optimization Toolkit to optimize for.<br><br><b>Default</b> - The Post-Training Optimization Toolkit's default | `--target_device`  | `./quantizer.py --all`<br>`--dataset_dir <DATASET_DIR>`<br>`--target_device VPU`<br><br>The supported values are those accepted <br>by the `target_device` option in the<br>Post-Training Optimization Toolkit's config files. |
| Print the quantization commands without running them.<br>This option creates the Post-Training Optimization Toolkit configuration file for you to inspect. | `--dry_run` | `./quantizer.py --all`<br>`--dataset_dir <DATASET_DIR> --dry_run` |


Also see the <a href="#shared-options">options applicable to all tools in this document</a>.

## How to use the Model Information Dumper

The basic Model Information Dumper command:

```sh
./info_dumper.py --all
```

This command prints iformation about all models to the standard output.

The script's output is a JSON array where each element is a JSON object that describes a single model. Each object has the following keys:


| Key | Description |  
|---|---|
| `name` | Mode identifier, as accepted by the `--name` option. | 
| `description` | Text describing the model. Paragraphs are separated by line feed characters. | 
| `framework` | A string that identifies the framework the format the model is downloaded in. Possible values are:<br> - `dldt` (Intermediate Representation)<br> - `caffe`<br> - `caffe2`<br> - `mxnet`<br> - `onnx`<br> - `pytorch`<br> - `tf` (TensorFlow)  | 
| `license_url` | An URL for the license under which the model is distributed. | 
| `precisions` | The list of precisions that the model has IR files for. For models downloaded in a format other than the Inference Engine IR format, these are the precisions that the model converter can produce IR files in. Possible values are:<br> - `FP16`<br> - `FP16-INT1`<br> - `FP16-INT8`<br> - `FP32`<br> - `FP32-INT1`<br> - `FP32-INT8` | 
| `subdirectory` | The subdirectory of the output tree into which the downloaded or converted files is placed by the downloader or the converter, respectively. | 
| `task_type` | A string identifying the type of task that the model performs. Possible values are:<br> - `action_recognition`<br> - `classification`<br> - `detection`<br> - `face_recognition`<br> - `feature_extraction`<br> - `head_pose_estimation`<br> - `image_inpainting`<br> - `image_processing`<br> - `image_translation`<br> - `instance_segmentation`<br> - `machine_translation`<br> - `monocular_depth_estimation`<br> - `object_attributes`<br> - `optical_character_recognition`<br> - `question_answering`<br> - `semantic_segmentation`<br> - `sound_classification`<br> - `speech_recognition`<br> - `style_transfer`<br> - `token_recognition`<br> - `text_to_speech`|  


Also see the <a href="#shared-options">options applicable to all tools in this document</a>.

## Options Applicable to all Tools in this Document<a name="shared-tools"></a>

All tools accept the following options. These options are mutually exclusive.

| Desired Result | Use Parameter |  Example Command  |
|---|---|---|
| View help. | `-h`<br>`--help` | `./TOOL.py --help` |
| Select all models. | `--all` | `./TOOL.py --all` |
| Select models that match a pattern that you  provide in a <br>comma-delimited list. The patterns can contain shell-style wildcards.<br><br>See https://docs.python.org/3/library/fnmatch.html for a full <br>description of the pattern syntax. | `--name` | `./TOOL.py`<br>`--name 'mtcnn-p,densenet-*'` |
| Use a file that contains a list of patterns to select models that contain<br>at least one of the patterns.<br>The file must contain at least one pattern per line. <br>Blank lines and comments starting with `#` are ignored. | `--list` | `./TOOL.py`<br>`--list <PATTERN-LIST>.lst`<br><br>Pattern list example:<br>`mtcnn-p densenet-* # get all` <br>`DenseNet variants` |
| See all model names defined in the configuration file. | `--print_all` | `./TOOL.py --print_all` |



__________

OpenVINO is a trademark of Intel Corporation or its subsidiaries in the U.S.
and/or other countries.


Copyright &copy; 2018-2021 Intel Corporation

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

     http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
