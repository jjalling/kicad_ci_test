# kicad_ci_test

![CI](https://github.com/INTI-CMNB/kicad_ci_test/workflows/CI/badge.svg)

.Test for Continuous Integration using KiCad.

This repo contains two examples, one is the simplest situation and the other complicates things a little bit.

## Simple example

This is an example of how to test and generate documentation for a KiCad project automatically every time you commit a change to the schematic and/or PCB.

In this example we use a [docker image](https://github.com/INTI-CMNB/kicad_auto) containing KiCad, [KiPlot](https://github.com/INTI-CMNB/kiplot) and all the supporting tools.

The [kicad_ci_test.kiplot.yaml](https://github.com/INTI-CMNB/kicad_ci_test/blob/master/kicad_ci_test.kiplot.yaml) configures KiPlot and says what's available.

The [.github/workflows/test1.yml](https://github.com/INTI-CMNB/kicad_ci_test/blob/master/.github/workflows/test1.yml) configures what GitHub makes.

In this file we:

* Instruct GitHub to run the tasks every time a change in the pertinent files is detected:

```
on:
  push:
    branches: [ master ]
    paths:
      - 'kicad_ci_test.sch'
      - 'kicad_ci_test.kicad_pcb'
  pull_request:
    branches: [ master ]
    paths:
      - 'kicad_ci_test.sch'
      - 'kicad_ci_test.kicad_pcb'
```

* Define actions for ERC, DRC, schematic documentation generation and PCB documentation generation.
* Each action takes the repo content as input using:

```
- uses: actions/checkout@v2
```

* The actions are executed by KiPlot and stored in the *Fabrication* directory

* The resulting files are stored as *artifacts* using:

```
 - name: Recuperar resultados
      uses: actions/upload-artifact@v1
      with:
        name: ERC_Output
        path: Fabrication
```

For more information about the GitHub configuration read the [Workflow syntax for GitHub Actions](https://help.github.com/en/actions/reference/workflow-syntax-for-github-actions) page.


## Second example

This example is a little bit more complicated. First we have our project located in a sub directory called *test_subdir* and second we use an external library found in the [kicad_ci_test_lib](https://github.com/INTI-CMNB/kicad_ci_test_lib) project.

The [test_subdir/test_subdir.kiplot.yaml](https://github.com/INTI-CMNB/kicad_ci_test/blob/master/test_subdir/test_subdir.kiplot.yaml) configures KiPlot and says what's available.

Note that this project has a four layers PCB so we generate gerbers for all of them. In the example file we refer to the ground plane as *GND.Cu* this is the name used in our PCB. Another option is to use the legacy KiPlot format: *Inner.1*, and this is what we do for the power plane (*Inner.2* in this case).

The [.github/workflows/test_subdir.yml](https://github.com/INTI-CMNB/kicad_ci_test/blob/master/.github/workflows/test_subdir.yml) configures what GitHub makes.

In this example we are putting all the real commands inside a *Makefile* and then invoking it like this:

```
    - name: Run ERC
      run: |
        make -C test_subdir/ erc
```

This changes the workind directory to *test_subdir/* and then makes the *erc* target, defined in the [Makefile](https://github.com/INTI-CMNB/kicad_ci_test/blob/master/test_subdir/Makefile) like this:

```
erc:
	kiplot -d Fabrication -s run_drc,update_xml -i
```

Using a Makefile is just another way of doing things, you could put the commands in the GitHub worflow, but don't forget to change the working directory to *test_subdir*.

Another detail is how to checkout the submodules. One option is to use the *submodules* option of the *checkout* action. But for this we need the *git* command, which isn't part of the default *kicad_auto* docker image.

To solve this we use an auxiliar job called *get_submodules*, this job runs on the host operating system, which includes *git*, and does a checkout using:

```
    - uses: actions/checkout@v2
      with:
        submodules: 'true'
```

So no we have the submodules. Note that if you need to get submodules inside submodules you'll need to use *recursive* instead of *true*.

Then this job creates an artifcat containig the submodule using:

```
    - name: Store submodule
      uses: actions/upload-artifact@v1
      with:
        name: Submodule
        path: test_subdir/lib
```

So then other jobs can get the needed files using:

```
    - name: Get submodule
      uses: actions/download-artifact@v2
      with:
        name: Submodule
        path: test_subdir/lib
```

In this way we get the submodules and we don't need to add *git* to *kicad_auto*.

