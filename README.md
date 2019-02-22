# Introducing C-PAC

## subtitle: An amazing tool many people have put a lot of time into, but is still difficult to get working on a user's machine

### Topics

- [C-PAC](http://fcp-indi.github.io/)
- [singularity](https://www.sylabs.io/guides/2.6/user-guide/)
- [nipype](https://nipype.readthedocs.io/en/latest/)
- [yaml](https://en.wikipedia.org/wiki/YAML)

### Prerequisites 

- [install singularity](https://www.sylabs.io/guides/2.6/user-guide/installation.html#installation)
- [download the test data](https://files.osf.io/v1/resources/fvuh8/providers/osfstorage/57f32a429ad5a101f977eb75)
- have some comfort working in a terminal environment

### What you will learn

1. How to Download CPAC with singularity
2. How to download/use the CPAC gui
3. Overview of the configuration file
4. How to theoretically run CPAC
5. How to debug issues


### Instructions

#### Installing the CPAC application

CPAC can be installed multiple ways, but I want to emphasize the usage of containers and how useful they can be.
I chose singularity because our cluster supports that software and it maintains the benefits of docker.
CPAC has a direct release from singularityhub, so we will be using that instead of converting a docker image to a singularity image (also see [this issue](https://github.com/FCP-INDI/C-PAC/issues/853)).

In your terminal, the command to install C-PAC via singularity is:

```bash
singularity pull shub://FCP-INDI/C-PAC
```

the output should be an image named: `FCP-INDI-C-PAC-master-latest.simg`

#### Installing the CPAC GUI

CPAC has a fresh [gui application](http://fcp-indi.github.io/docs/user/new_gui.html) you can install.
Follow the linked instructions to install.

#### Getting Configured

There are two configurations CPAC cares about:

1. the data configuration
2. the pipeline configuration

For this tutorial I'm assuming the data will be in BIDS format and we will not have to even consider the data configuration file (I still could not get that to work properly)

The pipeline configuration will be a little more in depth and require some manual editing of the files.
We will create our configuration file with the CPAC GUI which can be started like so:

```bash
c-pac_gui
```

I will not go into detail about the organization of the pipeline since [the documentation](http://fcp-indi.github.io/docs/user/pipeline_config.html) covers this well.
Overall, you can select what resources the command will have access to, how you want the anatomical images processed, how you want the functional images to preprocessed and analyzed.

I will say however, that the output paths under the **General** section will be ignored from our call of CPAC, so you can effective ignore those options.

There are several paths that do not resolve correctly in the GUI that you will see specified in the default view.

```bash
${environment.paths.fsl_dir}
${pipeline.anatomical.registration.resolution}mm
${pipeline.functional.template_registration.functional_resolution}
```

These should be replaced as follows:

```bash
${environment.paths.fsl_dir} ->  $FSLDIR
${pipeline.anatomical.registration.resolution}mm -> ${resolution_for_anat}
${pipeline.functional.template_registration.functional_resolution} -> ${resolution_for_func_preproc}
```

You can either do this inside the gui or after the config is created with a text editor (or via the terminal (see `sed`))

#### Running CPAC

Let's say we made a config file named `mah_pipes.yml`

Now we are ready to run CPAC (theoretically)

To run CPAC with singularity, I recommend to use the option `--cleanenv` since you may have FSL/freesurfer installed on your local machine and you do not want those ENVIRONMENT VARIABLES to contaminant your singularity container. 
If you exclude the `--cleanenv` option, you may get output that looks like this:

```python
Traceback (most recent call last):
  File "/code/run.py", line 14, in <module>
    from CPAC.utils.yaml import create_yaml_from_template
  File "/code/CPAC/__init__.py", line 21, in <module>
    import anat_preproc, \
  File "/code/CPAC/timeseries/__init__.py", line 1, in <module>
    from timeseries_analysis import create_surface_registration, \
  File "/code/CPAC/timeseries/timeseries_analysis.py", line 5, in <module>
    import nipype.interfaces.freesurfer as fs
  File "/usr/local/miniconda/lib/python2.7/site-packages/nipype/interfaces/freesurfer/__init__.py", line 7, in <module>
    from .preprocess import (
  File "/usr/local/miniconda/lib/python2.7/site-packages/nipype/interfaces/freesurfer/preprocess.py", line 33, in <module>
    FSVersion = Info.looseversion().vstring
  File "/usr/local/miniconda/lib/python2.7/site-packages/nipype/interfaces/freesurfer/base.py", line 56, in looseversion
    ver = cls.version()
  File "/usr/local/miniconda/lib/python2.7/site-packages/nipype/interfaces/base/core.py", line 1344, in version
    with open(klass.version_file, 'rt') as fobj:
IOError: [Errno 2] No such file or directory: '/usr/local/freesurfer/build-stamp.txt'
```

Another gotcha is that CPAC writes intermediate files in a root scratch directory, so you must bind the scratch directory to a local directory that you have permission to write to not get that error.
You can "bind" the scratch directory with this option: `-B /my/local/directory:/scratch`.
If you do not do this, you can get this error:

```python
Traceback (most recent call last):
  File "/code/run.py", line 498, in <module>
    tracking=not args.tracking_opt_out)
  File "/code/CPAC/pipeline/cpac_runner.py", line 493, in run
    raise Exception(err)
Exception: 

[!] CPAC says: Could not create the working directory: /scratch/working

Make sure you have permissions to write to this directory.
```

With those two gotchas in mind, an example call using singularity looks like this:

```bash
singularity run -B ${PWD}:/scratch --cleanenv FCP-INDI-C-PAC-master-latest.simg --pipeline_file mah_pipes.yml ${PWD}/ds005 ${PWD}/ds005/derivatives/cpac participant
```

#### Debugging when errors occur

Even with your best faith efforts to get everything set up correctly, you may still run into errors.
With any luck, nipype will give you a crash file to view (in a pklz format)

```bash
190222-17:44:12,292 nipype.workflow INFO:
	 crashfile: /home/james/cpacTest/ds005/derivatives/cpac/crash/crash-20190222-173835-james-frequency_filter_0.aI.a1.b0.c0.d0-20dd257a-15f6-4648-a8ba-5606ac8da9f6.pklz
```

To view the contents of this file we need the `nipypecli` command, but this command is only accessible through our singularity container. What can we do?

Thankfully `singularity exec` exists and allows us to access any command in the singularity container.

To observe the above file, we would type:

```bash
singularity exec --cleanenv FCP-INDI-C-PAC-master-latest.simg nipypecli crash /home/james/cpacTest/ds005/derivatives/cpac/crash/crash-20190222-173835-james-frequency_filter_0.aI.a1.b0.c0.d0-20dd257a-15f6-4648-a8ba-5606ac8da9f6.pklz
```

which will give us information on why the crash occurred so we can see what needs to be changed in the configuration file or if we need to [open an issue](https://github.com/FCP-INDI/C-PAC/issues)