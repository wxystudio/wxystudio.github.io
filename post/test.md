```diff
- I found this method and share it with the people I trust, please don't tell others, just use it by your own.
```

First I assume you have been an authorized chtc user, if not, please do it. https://chtc.cs.wisc.edu/uw-research-computing/form.html
And then, you must be familiar with the official example in https://github.com/CHTC/templates-GPUs/tree/master/conda/pytorch-1.9. You should know how to log in to your submitting node, how to run a job in standard process, and get output from running node. The above knowledge is beyond the scope of this blog, you should look at the official tutorial for reference.

# 1. Login guidance

the official related documentation is here:
https://chtc.cs.wisc.edu/uw-research-computing/connecting

![image](https://github.com/wxystudio/wxystudio.github.io/assets/17384283/773e1e65-f568-4242-815c-539b3a50c9f0)

![image](https://github.com/wxystudio/wxystudio.github.io/assets/17384283/24d9d2ff-5381-4abe-8f50-25c9c7a2a245)


For example:

if your NetID is xwang2696, hostname is ap2001.chtc.wisc.edu, then you can login to your account by:

```
ssh xwang2696@ap2001.chtc.wisc.edu
```

# 2. Three concepts in the chtc:
## (1). submitting node:
the /home/xxx directory, users should submit their job here. it looks like this:
![submit](https://user-images.githubusercontent.com/17384283/233470445-8de66b0a-0783-4fec-b443-7d690e8fc9c9.png)

## (2). staging node:
the /staging/xxx directory, users should store their data here. it looks like this:
![stage](https://user-images.githubusercontent.com/17384283/233470700-5ed7a625-56b2-4a33-b250-3a11b0b22510.png)

## (3). running node:
the machine we can run nvidia-smi to see the gpus. users' submitted job will be run here. it looks like this:
![run](https://user-images.githubusercontent.com/17384283/233470958-e9a03906-2cf7-4f3f-937f-f5f1e914521e.png)


# 3. Normal process to use chtc:
(1). submit job with source code in submitting node
(2). fetch data from staging node
(3). wait in the waitlist until your code is running in running node

# 4. A tricky method
just ssh to running node and run your code, which can significantly save time, you don't need to submit a job and wait in the waitlist. 
Below is a step by step tutorial:
## (1). Pipeline
The pipeline of this method is first submit a job in submitting node, the job can make the running node sleep, then ssh to the running node, fetch your code and data from staging node, finally you can run your code on the running node.

## (2). the files you need before submitting a job 
The official guidance of submitting job is at https://chtc.cs.wisc.edu/uw-research-computing/htcondor-job-submission

The requirement of GPU parameter is at https://chtc.cs.wisc.edu/uw-research-computing/gpu-jobs.html

After log in to your submitting node, eg: **/home/xwang2696/**, (suppose my id is xwang2696), create a folder named **/occupy/**, and create:
1. **occupy.sub**, the function of this file is to let the running node run a script (**occupy.sh**) and transfer 3 files(**occupy.py, setup.sh, studio.yaml**).

<details>

<summary>click to see the content</summary>

```python
universe = vanilla
executable = occupy.sh
output = ./log/$(Cluster).out
log = ./log/$(Cluster).log
error = ./log/$(Cluster).err

transfer_input_files = occupy.py, studio.yaml, setup.sh
should_transfer_files = YES
when_to_transfer_output = ON_EXIT

Requirements = (Target.HasCHTCStaging == true)

#require_gpus = (DriverVersion >= 11.1)
request_gpus = 1

+WantGPULab = true
+GPUJobLength = "long"

request_cpus = 1
request_memory = 16GB
request_disk = 50GB

queue 1
```

</details>

2. **occupy.sh**, the function is to run the occupy.py. Note that although the standard maximum use time is 7 days, we can use it for one month.(But I suggest you re-apply a new running node every 10 days, because it is illegal)
 
<details>

<summary>click to see the content</summary>

```shell
#!/bin/bash
# Following the example from http://chtc.cs.wisc.edu/conda-installation.shtml
# except here we download the installer instead of transferring it
# Download a specific version of Miniconda instead of latest to improve
# reproducibility
export HOME=$PWD
wget -q https://repo.anaconda.com/miniconda/Miniconda3-py39_4.10.3-Linux-x86_64.sh -O miniconda.sh
sh miniconda.sh -b -p $HOME/miniconda3
rm miniconda.sh
export PATH=$HOME/miniconda3/bin:$PATH
# Set up conda
source $HOME/miniconda3/etc/profile.d/conda.sh
hash -r
conda config --set always_yes yes --set changeps1 no

# Modify these lines to run your desired Python script
python occupy.py 

```

</details>


3. **occupy.py**, the function is to let the running node sleep for one month.

<details>

<summary>click to see the content</summary>

```python
import time
time.sleep(3600000)

```

</details>



4. **setup.sh**, the function is to setup anaconda environment in running node.

<details>

<summary>click to see the content</summary>
```shell
source $HOME/miniconda3/etc/profile.d/conda.sh
find_in_conda_env(){
    conda env list | grep "${@}" >/dev/null 2>/dev/null
}
if find_in_conda_env ".*studio.*" ; then
   conda activate studio
else
        conda env create -f studio.yaml
        conda activate studio
fi
```


</details>


5. **studio.yaml**, this is a conda environment I create for myself, you can use your own conda environment instead.

<details>

<summary>click to see the content</summary>
```shell
name: studio
channels:
  - pytorch
  - nvidia
  - defaults
dependencies:
  - _libgcc_mutex=0.1=main
  - _openmp_mutex=5.1=1_gnu
  - blas=1.0=mkl
  - bottleneck=1.3.5=py39h7deecbd_0
  - bzip2=1.0.8=h5eee18b_6
  - ca-certificates=2024.3.11=h06a4308_0
  - cudatoolkit=11.1.74=h6bb024c_0
  - ffmpeg=4.2.2=h20bf706_0
  - freetype=2.12.1=h4a9f257_0
  - giflib=5.2.1=h5eee18b_3
  - gmp=6.2.1=h295c915_3
  - gnutls=3.6.15=he1e5248_0
  - intel-openmp=2023.1.0=hdb19cb5_46306
  - joblib=1.4.2=py39h06a4308_0
  - jpeg=9b=h024ee3a_2
  - lame=3.100=h7b6447c_0
  - lcms2=2.12=h3be6417_0
  - ld_impl_linux-64=2.38=h1181459_1
  - libffi=3.4.4=h6a678d5_1
  - libgcc-ng=11.2.0=h1234567_1
  - libgfortran-ng=11.2.0=h00389a5_1
  - libgfortran5=11.2.0=h1234567_1
  - libgomp=11.2.0=h1234567_1
  - libidn2=2.3.4=h5eee18b_0
  - libopus=1.3.1=h7b6447c_0
  - libpng=1.6.39=h5eee18b_0
  - libstdcxx-ng=11.2.0=h1234567_1
  - libtasn1=4.19.0=h5eee18b_0
  - libtiff=4.1.0=h2733197_1
  - libunistring=0.9.10=h27cfd23_0
  - libuv=1.44.2=h5eee18b_0
  - libvpx=1.7.0=h439df22_0
  - libwebp=1.2.0=h89dd481_0
  - lightning-utilities=0.9.0=py39h06a4308_0
  - lz4-c=1.9.4=h6a678d5_1
  - mkl=2023.1.0=h213fc3f_46344
  - mkl-service=2.4.0=py39h5eee18b_1
  - mkl_fft=1.3.8=py39h5eee18b_0
  - mkl_random=1.2.4=py39hdb19cb5_0
  - ncurses=6.4=h6a678d5_0
  - nettle=3.7.3=hbbd107a_1
  - ninja=1.10.2=h06a4308_5
  - ninja-base=1.10.2=hd09550d_5
  - numexpr=2.8.4=py39hc78ab66_1
  - numpy=1.20.3=py39h7820934_1
  - numpy-base=1.20.3=py39h7e635b3_1
  - openh264=2.1.1=h4ff587b_0
  - openssl=3.0.14=h5eee18b_0
  - packaging=23.2=py39h06a4308_0
  - pandas=1.5.3=py39h417a72b_0
  - pillow=9.3.0=py39hace64e9_1
  - pip=24.0=py39h06a4308_0
  - python=3.9.19=h955ad1f_1
  - python-dateutil=2.9.0post0=py39h06a4308_2
  - pytorch=1.9.1=py3.9_cuda11.1_cudnn8.0.5_0
  - pytz=2024.1=py39h06a4308_0
  - readline=8.2=h5eee18b_0
  - scikit-learn=1.1.3=py39h6a678d5_1
  - scipy=1.9.3=py39hf6e8229_2
  - setuptools=69.5.1=py39h06a4308_0
  - six=1.16.0=pyhd3eb1b0_1
  - sqlite=3.45.3=h5eee18b_0
  - tbb=2021.8.0=hdb19cb5_0
  - threadpoolctl=2.2.0=pyh0d69192_0
  - tk=8.6.14=h39e8969_0
  - torchmetrics=1.1.2=py39h06a4308_1
  - torchvision=0.10.1=py39_cu111
  - tqdm=4.66.4=py39h2f386ee_0
  - typing_extensions=4.11.0=py39h06a4308_0
  - tzdata=2024a=h04d1e81_0
  - wheel=0.43.0=py39h06a4308_0
  - x264=1!157.20191217=h7b6447c_0
  - xz=5.4.6=h5eee18b_1
  - zlib=1.2.13=h5eee18b_1
  - zstd=1.4.9=haebb681_0
prefix: /var/lib/condor/execute/slot1/dir_3284020/miniconda3/envs/pytorch
```


</details>


## (3). How to submit a job and require a running machine
As the figure below, we need to create a folder **/occupy/**, put all the above files into it, and run:
```shell
condor_submit occupy.sub
```
to submit this job. It looks like this:
![s](https://user-images.githubusercontent.com/17384283/233472344-5f7a0e01-6ec0-46eb-a3d0-d3e9c024f593.png)

## (4). How to view all your running jobs and access them
run:
```shell
condor_q
```
to look at the id of your job
then run:
```shell
condor_ssh_to_job id
```
to ssh into your running node. It looks like this:
![r1](https://user-images.githubusercontent.com/17384283/233473249-6c4d8193-b357-46a2-a6aa-bafa679b3fc6.png)

## (5). When you access into the running machine
Now you will be in the running node, you can run:
```shell
nvidia-smi
```
to see the availble gpus. It looks like this:
![run](https://user-images.githubusercontent.com/17384283/233472617-28b1b2c1-ec46-45b3-bb8e-e0c32bca5f18.png)

or run:
```shell
ls
```
to see the files we transfer from submitting node. It looks like this:
![t](https://user-images.githubusercontent.com/17384283/233472710-e0da585e-7218-4ed1-b2e6-7253214ad71e.png)
You can see here are 4 files **occupy.py, wxy.yaml, setup.sh, filetrans.sh**, we have uploaded them by our script.

## (6). Some common setup processes before using running machine (Anaconda)
**Every time** you ssh into the running node, you need to run:
```shell
source setup.sh
```
to activate your conda environment, I don't know why, But it is required.

## (7). How to fetch your code and data , run your code in the running machine
It's almost down! 
you can transfer your code from staging node **/staging/xwang2696/** like:
```shell
rsync -avP /staging/xwang2696/code ./
```
then you can run:
```shell
python code.py
```
to run your code!

# 4. How to run your code and debug: general idea
First, I find that we **can only transfer files between staging node and running node**
you can transfer your data and code by:

```shell
rsync -avP /staging/xwang2696/code/BNN ./
```

## (1). local debug and upload to run
You can debug on your local machine, then upload it to staging node by rsync, then fetch your code from staging node to running node, then run it.

## (2). vim
If you are a vim master, you can just edit your code in the running node with vim.

## (3). staging node debug and upload to run
But I suggest you write code in staging node **/staging/xwang2696/**, then upload it to the running node for running. The advantage is obvious, we can use vscode remote to connect the staging node, and we can use copilot, which is powerful for coding. 

# 5. some useful commands and tips for chtc use:

## (1). When you are in chtc

<details>

<summary>click to see the content</summary>


```shell
# submit a job
condor_submit xxx.sub

# view all your jobs
condor_q

# delete job
condor_rm

# access to your job
condor_ssh_to_job xxx

# transfer your data between running node and staging node
rsync -avP to staging

# see your quotas in staging node
get_quotas /staging/xwang2696

# see your quotas in submitting node
quota -vs

# count your file numbers in this folder
ls -lR| grep "^-" | wc -l
```

</details>


## (2). After ssh into the server:
```shell
source $HOME/miniconda3/etc/profile.d/conda.sh 
conda env create -f wxy.yaml
python xxx.py

condor_ssh_to_job xxx
conda env create -f wxy.yaml
source $HOME/miniconda3/etc/profile.d/conda.sh 
```

