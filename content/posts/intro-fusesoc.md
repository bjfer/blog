+++
title = "Trying FuseSoc"
date = 2025-08-05T22:53:00+02:00
tags = ["vhdl"]
draft = false
+++

I recently decided to try out [FuseSoC](https://github.com/olofk/fusesoc) in one of my FPGA work projects to see how easy the migration would be. The following are my impressions of using the tool, without going into detail about FuseSoC and Edalize; my suggestion would be that you check the documentation and then read this post to get to know a practical example.

My project targets a 10x ADC digitizer board which includes a Kintex Untrascale for data processing, the [SIS8300Ku](https://www.struck.de/sis8300-ku.html). The board also includes 2x DACs, exernal clock and trigger interfaces, 2Gb of DDR4 memory, among other interfaces. Different applications (data processing logic) is available for specific experiments and setups, while the board logic remains the same, i.e. the code that allows configuration and monitorization of the hardware features. A `project`, in this context, is the combination of a particular `application module` and the common `board module`, a fairly common scenario.

All `applications` must have the `timingInfo` module; it decodes timing related information and includes generics to generate or exclude logic (again, depends on the application).

{{< figure src="/images/fpga.png" >}}

With this scenario in mind, I wanted to update my project in the following manner:

-   declare the `timingInfo` as a remote core. Currently, the module is included as a git submodule
-   have a core file to synthesize the `board module`, separated from the final project, to generate a _Design Checkpoint_ file. This file should then be included in all projects, saving compilation time
-   have a single core file for all applications; each target will be a FPGA project, i.e the `board logic`, the files for a specific `application logic` and the `timingInfo` configure with the proper `generics` values for that application

{{< figure src="/images/fuse.png" >}}


## Remote core {#remote-core}

After creating a `fusesoc.conf` file in the root folder of my project, adding the `timingInfo` module as a remote core was straightforward:

```shell
fusesoc library add timingInfo gitLinkToModule
```

The proper lines will be added to the `fusesoc.conf` file; since I needed to add a `.core` file to the project, I created a new branch, _test/fusesoc_. To make sure that the tool downloads the right branch, the property **sync-version** needed to be added to the `fusesoc.conf` together with the branch name (could have been the sha value as well).

> [library.timingInfo]
>
> location = fusesoc_libraries/timingInfo
>
> sync-uri = gitLinkToModule
>
> sync-type = git
>
> **sync-version = test/fusesoc**

The `timingInfo.core` _YAML_ file is shown below. For now the only target is the **default**, since this is the one that is used when <span class="underline">referencing the core within another top core</span>. This means that all files in the rtl set and the defined generics will be available in the top's core target that uses timingInfo as a dependency.

```yaml
CAPI=2:
name: euxfel:common:timingInfo:1.8.2
description: TimingInfo module

filesets:
   rtl:
      files:
         - files1.vhd
         - src/file2.vhd
         - ...
      file_type: vhdlSource-2008

targets:
   default: &default
   filesets:
      - rtl
   parameters:
      - generic1=true
      - generic2=3
      - ...

parameters:
   generic1:
      datatype    : bool
      description : Enable feature 1
      paramtype   : generic
   generic2:
      datatype    : int
      description : Size of output 2
      paramtype   : generic
   ...
```

From this simple core, a couple of first impressions:

-   each source must be specifically mention, there is no use of regular expressions. This can be seen as quite annoying or a fail-safe: only the required sources are added to the proper list.
-   the sync-version property is **not** mention in the documentation, I happen to find another FuseSoC project that used it. Running the command _fusesoc library add --help_ would also show that the option.
-   the property _name_ follows the VLNV convention, introducing yet another version number. In our case, git's _sha_ would be enough, and the module itself includes a hardware coded version, so we now have three version numbers to maintain. The information <span class="underline">is optional</span>, however when taking into account that only the default target is read when sourcing a core file, the role of FuseSoC's version could be used in these example scenarios:
-   entities with multiple architectures (and sources associated with the latter)
-   different sources depending on specific FPGA platforms (this could also be covered [using flags](https://fusesoc.readthedocs.io/en/stable/user/build_system/flags.html#using-flags))
-   manage exposed generics to the top module


## Generating the board checkpoint file {#generating-the-board-checkpoint-file}

Since my `.conf` file is saved in the root directory of the project, I need to update it so that it also searches for core files in that directory. This is easily done running

```shell
fusesoc library add --sync-type local sis8300ku .
```

The .conf file will then be updated to include the following lines:

> [library.sis8300ku]
>
> location = /full/path/to/project
>
> sync-uri = .
>
> sync-type = local
>
> auto-sync = true

I grouped the different type of files of the board into filesets in `board.core` file.

```yaml
CAPI=2:

name: euxfel:sis8300ku:board:0.20.14
description: SIS8300 KU board logic

filesets:
   rtl:
      files:
         - sources/hdl/file1.vhd
         - sources/hdl/file2.vhd:
         file_type: vhdlSource
         ...
      file_type: vhdlSource-2008
  configProj:
      files:
         - scripts/vivadoProject.tcl
         - sources/bd/pcie_ddr.tcl
         - scripts/bdWrapper.tcl
      file_type: tclSource
   genBoardCheckpoint:
      files:
         - scripts/genCheckpoint.tcl:
         file_type: user
         - scripts/genCheckpointCall.tcl
      file_type: tclSource
   boardCheckpoint:
      files:
         - sources/dcp/board.dcp:
         file_type: user
         - scripts/addDcp.tcl:
      file_type: tclSource
```

A couple of new concepts are show:

-   all rtl files are define as `vhdl-08` sources except `file2.vhd`, but there is no need to create a separate fileset.
-   the `vivadoProject.tcl` script configures the vivado tool to my project. Some of these options include VHDL as the target language and use a specific folder as the source of user IPs. Specifying it as a `tclScript` file makes sure that vivado will source it when creating the project.
-   a `user` file type should not be imported by the tool (in my case vivado) but must be included in the build directory that FuseSOC will create. You can use the _--no-export_ flag in FuseSoC so that sources are not copied into the build directory - this is my prefer case, as to avoid situations where I start fixing the copy.
-   since the checkpoint point is not a default artifact of _Edalize's_ _build_, the `genBoardCheckpoint` fileset includes the `genCheckpointCall.tcl` source that will call the user file `genCheckpoint.tcl`. The latter must be added in case another FuseSoC user does not use the _--no-export_ flag. The tcl commands to generate the checkpoint file are:

> set scriptLocation [file normalize [info script]]
>
> set_property STEPS.SYNTH_DESIGN.TCL.POST "[file dirname $scriptLocation]/genCheckpoint.tcl" [get_runs synth_1]
-   finally, the `boardCheckpoint` includes the generated dcp file and a script to read it in a vivado project since FuseSoC does not recognize .dcp files. This fileset will be my default target, i.e., these should be the only files imported on my final FPGA project.

The rest of the YAML file is as follows:

```yaml
targets:
   default: &default
   filesets:
      - boardCheckpoint
   hooks:
      post_build: [saveMaps]

fastAdcBoard:
   filesets:
      - configProj
      - rtl
      - bd
      - genBoardCheckpoint
   toplevel: ent_board
   description: Generate dcp file for SIS8300ku board
   default_tool: vivado
   tools:
      vivado:
      part: xcku040-ffva1156-1-c
      pnr: none
      source_mgmt_mode: All
   # EDA flow API does not yet support pnr
   # flow: vivado
   # flow_options:
   #     part: xcku040-ffva1156-1-c
   #     pnr: none
   #     source_mgmt_mode: All
   #Save artifacts
   hooks:
      post_build: [saveMaps]

scripts:
   saveMaps:
      cmd: ['cp','*.maps','../../../dist/maps/']
```

The `fastAdcBoard` target generates the desired checkpoint file; with `pnr: none,` Edalize will stop the build right after the synthesis. This option is not yet supported in the new Edalize flow API, hence I am using the legacy tool API. There are also custom maps files generated by an internal board module which should be saved; a `hook` is defined that should run after build is completed.

Running FuseSOC with the _--setup_ flag will create the build folder, tcl scripts and Makefile file used later by the vivado tool to create, compile and run the project. This is very convenient for debugging, experiment (change tcl scripts created, create new Makefile targets, etc.) or just create the project to then continue in vivado. It also made it clear that the _post_build_ hook defined in my core file would never run, since Edalize's build **always assumes that a bit file is generated**.

Edalize supports [a great number of tools](https://edalize.readthedocs.io/en/latest/edam/api.html#tool-options), however the list of options per tool is quite niche. I was surprised that simple parameters, like the target language in vivado, is not readily available. This is to say, be prepared to write some simple scripts for your tool when migrating to FuseSoC.


## The project core file {#the-project-core-file}

The last core file uses the previous two as dependencies, and each target defines a bit file that integrates a different application, i.e. a collection of source files for specific data processing.

```yaml
CAPI=2:

name: euxfel:sis8300ku:applications:0.9.1
description: SIS8300 KU applications

filesets:
   topDesign:
      files:
         - sources/hdl/TOP_SIS8300_KU.vhd
         - sources/hdl/PKG_UTILS.vhd
         - sources/constraints/board_pins.xdc:
         file_type: xdc
         - sources/constraints/board_fpga_constrains.xdc:
         file_type: xdc
         - scripts/vhdlProject.tcl:
         file_type: tclSource
         # Do not use xdc files during synthesis
         - scripts/constraints.tcl:
         file_type: tclSource
      file_type: vhdlSource-2008
      depend:
         - euxfel:sis8300ku:board:0.20.14
         - ">=euxfel:common:timingInfo:1.8"
```

The `depend` parameter is where I reference the other two core files; this will import all sources, the generics and any defined hooks from both `default` targets of the `timingInfo` and `board` to this fileset. Just as an example, I specify that the board core must have a specific version, while the `timingInfo` must be above or equal to 1.8.

```yaml
app1:
   files:
      - sources/hdl/application_1/file1.vhd
      - ...
      - sources/ip/application_1/clk_wiz_dac.xci:
      copyto: ip/clk_wiz_dac/clk_wiz_dac.xci
      file_type: xci
      - sources/ip/application_1/c_shift_ram_0.xci:
      copyto: ip/clk_wiz_dac/clk_wiz_dac.xci
      file_type: xci
   file_type: vhdlSource-2008
```

This is an example of an application fileset which introduces the `copyto` parameter. In my project folder, all application specific IPs are under a single folder, however vivado requires that each `xci` must be in a separate folder when these sources are read (not added/imported) into a project. The `copyto` takes care of this, specifying the path (inside the build's folder) where the file should be, creating any required subfolder.

```yaml
targets:
   # The "default" target is the top module
   default: &default
   filesets:
      - topDesign
   toplevel: ent_sis8300_ku
   flow: vivado
   flow_options:
      part: xcku040-ffva1156-1-c
      source_mgmt_mode: All
   parameters:
   # Default values for timing
      - generic1=false
      - generic2=1

app1:
   <<: *default
   filesets_append:
      - app1
   description: Application 1

app2:
   <<: *default
   filesets_append:
      - app2
   description: Application 2
   parameters:
      - generic1=true
      - generic2=2
```

The app targets are very simple and straightforward. If necessary, I could override any of the parameters values of the imported default target, as it is done in `app2` for the timingInfo module. If I were to target a different tool, FuseSOC's makes it very easy to [include extra flags](https://fusesoc.readthedocs.io/en/stable/user/build_system/flow_options.html) in the _flow/flow_definitions_ default target definition or [via command line](https://fusesoc.readthedocs.io/en/stable/user/cli.html) when calling the tool.

Custom maps are also generated from the apps; the `hook` included in the `fastAdcBoard` default target will take care of copying it to the proper folder.

There is no dependency between targets, so the final project Makefile looks like this:

```shell
#Get core name
define core_name
$(shell sed -n -e 's/^name: \(.*\)/\1/p' $(1))
endef

#Replace in core name : with _
define core_name_folder
$(subst :,_,$(1))
endef

FUSESOC := python/environment/fusesoc/bin/fusesoc
BOARD_VLNV := $(call core_name, fastAdcBoard.core)
BOARD_NAME := $(call core_name_folder, $(BOARD_VLNV))
APPS_VLNV := $(call core_name, application.core)
APPS_NAME := $(call core_name_folder, $(FASTADC_VLNV))

DIST_FOLDER := dist

.PHONY: clean
clean:
@echo "Deleting contents of the build directory"
@rm -rf $(BUILD)

# Generate board checkpoint
fastAdcBoard:
   $(FUSESOC) run --clean --build --no-export --target $@ $(BOARD_VLNV)
   # FuseSOC does not run post_build hooks if target does not generate a bit file
   cp build/${BOARD_NAME}/$@/*.maps ${DIST_FOLDER}/maps

app1: fastAdcBoard
   $(FUSESOC) run --clean --build --no-export --target $@ $(APPS_VLNV)
   #TODO add hook in default target to copy bit file
   cp build/${APPS_NAME}/$@/*.bit ${DIST_FOLDER}/firmware/
```


## Final remarks {#final-remarks}

There are ton of features from FuseSOC's that were out of my project scope (filters, generators, virtual cores and mappring). Nevertheless, the overall experience is a very positive one. The YAML file really makes the flow clear and very readable, while taking advantage of the format to easily reconfigure/create new projects. The documentation is enough to get simple projects up and running, but many things become more clear by digging into the source files in the git repository.
