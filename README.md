# RECAST for the VBF+MET Analysis #



## Main Developer

* Othmane Rifki (othmane.rifki@desy.de)

## Summary

Repository with the VBF+MET workflow needed to pass a signal DAOD through the analysis chain in order to make an interpretation and set a limit on it. The workflow is set up in such a way that i can run entirely on a local machine or the gitlab CI.

## Prerequisites

To run the workflow on your machine, the following prerequisites are needed:

1. A running [docker](https://docs.docker.com/install/) installation.
2. A running `RECAST` installation:  `pip install recast-atlas`
3. Optional: A [yadage](https://yadage.readthedocs.io/en/latest/introduction.html) installation. Alternatively, yadage commands (eg. packtivity-run, yadage-run, etc.) can be run from inside the official `yadage` docker container by executing the following in the terminal in which you plan to run them:
```bash
docker run --rm -it -e PACKTIVITY_WITHIN_DOCKER=true -v $PWD:$PWD -w $PWD -v /var/run/docker.sock:/var/run/docker.sock yadage/yadage sh
docker login gitlab-registry.cern.ch -u [your CERN username]
```
You will be asked for your CERN password.

There are two modes of running:
1. RECAST workflow full analysis chain
2. RECAST workflow test separate parts of the analysis chain
3. Docker containers without RECAST

## 1. RECAST workflow full analysis chain
Clone the workflow repository as follows:

```bash
git clone https://:@gitlab.cern.ch:8443/VBFInv/recast-vbfmet.git
```
You will be asked for your CERN credentials to clone the repo.
You can create a personal token `https://docs.gitlab.com/ee/user/profile/personal_access_tokens.html`

``` bash
export RECAST_AUTH_USERNAME=[your CERN username]
export RECAST_AUTH_PASSWORD=[your CERN password]
export RECAST_USER=[your CERN username]
export RECAST_PASS=[your CERN password]
export RECAST_TOKEN=[your Gitlab personal token]

eval "$(recast auth setup -a $RECAST_USER -a $RECAST_PASS -a $RECAST_TOKEN -a default)"
eval "$(recast auth write --basedir authdir)"

cd recast-vbfmet
$(recast catalogue add $PWD)
cd ..
recast catalogue ls
```
Now everything is setup to run the full workflow. To run it, do:
``` bash
recast run atlas/atlas-vbfmet --tag myrun
```

## 2. RECAST workflow test separate parts of the analysis chain
If you want to run seperate stages of the analyis, you can use the 3 tests i have setup.
You need to change the path to the input files. In my examples, i have the folder `input_files` where I put a DAOD, MiniNtuple, and a HistFitter example inputs.

To run the tests, do:

``` bash
$(recast tests run atlas/atlas-vbfmet  test_stanalysiscode --backend docker --tag stanalysiscode)
$(recast tests run atlas/atlas-vbfmet  test_stpostprocessing --backend docker --tag stpostprocessing)
$(recast tests run atlas/atlas-vbfmet  test_limitsetting --backend docker --tag limitsetting)
```
To run the tests interactively, do:
``` bash
$(recast tests shell atlas/atlas-vbfmet  test_stanalysiscode --backend docker --tag stanalysiscode)
$(recast tests shell atlas/atlas-vbfmet  test_stpostprocessing --backend docker --tag stpostprocessing)
$(recast tests shell atlas/atlas-vbfmet  test_limitsetting --backend docker --tag limitsetting)
```

## 3. Docker containers without RECAST

First, log in to docker to pull the docker images used in the workflow steps to your computer:

```bash
docker login gitlab-registry.cern.ch -u [your CERN username]
```
You will be asked for your CERN password. The images are then pulled as follows:
```bash
docker pull [docker image name]
```
Where the docker image names to pull are:
  * gitlab-registry.cern.ch/vbfinv/stanalysiscode:latest
  * gitlab-registry.cern.ch/vbfinv/stpostprocessing:master
  * gitlab-registry.cern.ch/vbfinv/statstools:recast3-4b143c38

a - STAnalysisCode
``` bash
 docker run --rm -it -w $PWD -v $PWD:$PWD gitlab-registry.cern.ch/vbfinv/stanalysiscode:latest bash
 export USER=othrif
 source ~/release_setup.sh
 runVBF.py -f root://eosatlas.cern.ch//eos/atlas/atlascerngroupdisk/phys-exotics/jdm/vbfinv/RECAST/input/DAOD_EXOT5/mc16_13TeV.346600.PowhegPy8EG_NNPDF30_AZNLOCTEQ6L1_VBFH125_ZZ4nu_MET75.deriv.DAOD_EXOT5.e7613_s3126_r9364_p3895/DAOD_EXOT5.18795494._000010.pool.root.1
```
b - STPostProcessing
``` bash
 cd /Users/othmanerifki/vbf/workflow/
 docker login gitlab-registry.cern.ch
 docker run --rm -it -w $PWD -v $PWD:$PWD gitlab-registry.cern.ch/vbfinv/stpostprocessing:master bash
 export DISPLAY=localhost:0.0 # needed to open root
 source ~/release_setup.sh
 source /analysis/build/*/setup.sh
 cd stpp/
 mkdir -p input_files/user.othrif.vXX.999999.PowhegPy8EG_NNPDF30_AZNLOCTEQ6L1_VBFH125_ZZ4nu_MET75.root
cp -r ../stac/submitDir/data-MiniNtuple/mc16_13TeV.346600.PowhegPy8EG_NNPDF30_AZNLOCTEQ6L1_VBFH125_ZZ4nu_MET75.root input_files/user.othrif.vXX.999999.PowhegPy8EG_NNPDF30_AZNLOCTEQ6L1_VBFH125_ZZ4nu_MET75.root/.
echo "$PWD/input_files/user.othrif.vXX.999999.PowhegPy8EG_NNPDF30_AZNLOCTEQ6L1_VBFH125_ZZ4nu_MET75.root" > list
python /analysis/src/VBFAnalysis/util/getN.py -l list -o fout.root
athena VBFAnalysis/VBFAnalysisAlgJobOptions.py --filesInput input_files/user.othrif.vXX.999999.PowhegPy8EG_NNPDF30_AZNLOCTEQ6L1_VBFH125_ZZ4nu_MET75.root/mc16_13TeV.346600.PowhegPy8EG_NNPDF30_AZNLOCTEQ6L1_VBFH125_ZZ4nu_MET75.root - --currentVariation Nominal - --normFile fout.root
mkdir -p output_files
hadd output_files/VBFH125.root  VBFH125*.root
mkdir -p HF_files
cd HF_files
athena VBFAnalysis/HFInputJobOptions.py --filesInput ../output_files/VBFH125.root - --currentVariation Nominal --Binning 11 --extraVars 7
hadd HFVBFHAltSignalNominal.root HFVBFHAltSignalNominal*root
```
c - Limit Setting
``` bash
docker run --rm -it -w $PWD -v $PWD:$PWD gitlab-registry.cern.ch/vbfinv/statstools:recast3-4b143c38 bash
source ~/release_setup.sh
cd /StatsCode/makeHistFitterConfig/HistFitterTutorial
source setup.sh
cd src/
make clean && make
cd ../..
source runStatOnlyBlindedDocker.sh exampleFile/HF_jan7_mc16all_nom.root HF_jan7_mc16all_nom
```
