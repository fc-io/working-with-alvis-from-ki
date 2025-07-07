# Working with Alvis for researchers at the Karolinska Institute.

Hopefully, helpful info for AI researchers at KI.

How to get started developing agents and doing inference on the Alvis supercomputer.

This is for doing variable interactive development. If you need long running jobs, or have a set use case then use more scripts and sbatch.

## Get Access

Apply at <https://supr.naiss.se/>.

You must be a PhD student or above – otherwise ask your supervisor to apply on your behalf.

## Caveats

### Working remotly (VPN)

When ssh-ing to alvis you need to be on Sunet. You are automatically on Sunet on the KI wired network or eduroam. 

However, when working from home you need a suitable VPN. Sadly the KI VPN won't help you out. What you can do instead is to use the Chalmers credentials provided to you when getting access to Alvis to connect through chalmers VPN (eduVPN).

### GPUs on Alvis

Alvis, sadly, don't have any Hopper or Blackwell GPUs so try to get acess to Berzelius if you can. Performance and setup will be a lot easier on GPUs of a newer generation than Ampere.

### Stick to Bash

Alvis are happy to provide zsh and fish but unless you know what you are doing stick to bash. The interaction between shells, slurm, containers etc is annonying to keep track of and fix.

## Gettings started

## local ssh config

TODO: add alvis to ~/.ssh/config

TODO: setup credentials

## remote setup


## Check for available GPU nodes

```bash
sinfo --Node --states=idle,mix --noheader \
      -O NodeList:12,StateCompact:8,Gres:90,GresUsed:90,CPUs:21,Memory:10 |
awk 'BEGIN { printf "%-12s %-8s %-35s %-35s %21s %10s\n", \
                   "NODE","STATE","GRES","GRES-USED","CPUs","MEM(MiB)" }
     $2!="plnd" && $3~/gpu:A100/ {
                   sub(/\(.*/,"",$3); sub(/\(.*/,"",$4);
                   printf "%-12s %-8s %-35s %-35s %21s %10s\n", \
                          $1,$2,$3,$4,$5,$6 }' |
column -t
```

```bash
# example:
$ sinfo [as above...]
NODE       STATE  GRES           GRES-USED      CPUs  MEM(MB)
alvis3-01  idle   gpu:A100:4     gpu:A100:0     64    512000
alvis3-02  mix    gpu:A100:4     gpu:A100:1     64    512000
alvis3-03  idle   gpu:A100:4     gpu:A100:0     64    512000
alvis3-04  idle   gpu:A100:4     gpu:A100:0     64    512000
alvis3-13  mix    gpu:A100:4     gpu:A100:3     64    256000
alvis3-15  mix    gpu:A100:4     gpu:A100:3     64    256000
alvis3-19  mix    gpu:A100:4     gpu:A100:2     64    256000
alvis3-21  mix    gpu:A100:4     gpu:A100:3     64    256000
alvis3-23  mix    gpu:A100:4     gpu:A100:1     64    256000
alvis3-35  mix    gpu:A100:4     gpu:A100:3     64    256000
alvis3-36  mix    gpu:A100:4     gpu:A100:2     64    256000
alvis3-37  mix    gpu:A100:4     gpu:A100:2     64    256000
alvis3-39  mix    gpu:A100fat:4  gpu:A100fat:4  64    1024000
alvis3-41  mix    gpu:A100fat:4  gpu:A100fat:3  64    1024000
...
```

## Get a node with one GPU

### Get your project id

First, get your project id. You can use `projinfo` to find it if you don't already know it.

```bash
projinfo
```

```bash
#example
$ projinfo
 Project                Used[h]         Allocated[h]      Queue
    User
---------------------------------------------------------------
NAISS1234-prj-id          18.97                  250      alvis
    my_user               18.97   
```

### Get an interactive allocation

```bash
salloc -A <REPLACE_WITH_YOUR_PROJECT_ID> -N1 --gres=gpu:A100:1 --time=3:00:00
```

### shell into the allocated node (don't ssh into the node as it can mess with the slurm scheduler if you have not yet tied up the allocation)
```bash
srun --pty bash -l 
```
### if you want an additional connection from another node
#### first, quick check, to see all running jobs

```bash
squeue -u $USER -O jobid,state,nodelist
```

```bash
#example:
# squeue -u $USER -O jobid,state,nodelist
JOBID               STATE               NODELIST            
4732902             COMPLETING          alvis3-35           
```

#### then use srun to connect to your instance/job
```bash
srun --jobid=<job_id> --overlap --pty bash -l 
```

### check that you have the right allocation
```bash
nvidia-smi
```
```bash
#example
$ nvidia-smi
Mon Jul  7 09:03:26 2025       
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 575.57.08              Driver Version: 575.57.08      CUDA Version: 12.9     |
|-----------------------------------------+------------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
|                                         |                        |               MIG M. |
|=========================================+========================+======================|
|   0  NVIDIA A100-SXM4-80GB          On  |   00000000:31:00.0 Off |                    0 |
| N/A   26C    P0             58W /  500W |       0MiB /  81920MiB |      0%      Default |
|                                         |                        |             Disabled |
+-----------------------------------------+------------------------+----------------------+
                                                                                         
+-----------------------------------------------------------------------------------------+
| Processes:                                                                              |
|  GPU   GI   CI              PID   Type   Process name                        GPU Memory |
|        ID   ID                                                               Usage      |
|=========================================================================================|
|  No running processes found                                                             |
+-----------------------------------------------------------------------------------------+

#0/81920MiB indicate that we have a a100fat allocated and ready to use
```

## Run inference
### download/setup your dev environment

TODO: add an example of downloading models and containers

### cd to container location

Alvis has a storage solution called Mimer that is avaiable on both login nodes and the GPU nodes.

```bash
cd /mimer/NOBACKUP/groups/<project_id>
```

### start container
```bash
apptainer shell --nv --bind <path_to_models_on_host>:<path_to_models_in_container> <name_and_path_to_appcontainer>
```

```bash
#example:
$ apptainer shell --nv --bind ./models:/models vllm-openai_latest.sif
```

### start vllm with tool use and tool auto support

TODO: add a multi node example

```bash
# example:
$ vllm serve ./models/Qwen3-32B-FP8  --enable-auto-tool-choice --tool-call-parser hermes --reasoning-parser deepseek_r1 --host 0.0.0.0 --port 8000
```

### after "INFO: Application startup complete." you can test by curling the http server from the host of the container

but you can also skip this and test from your local machine

``` bash
curl http://$(hostname -I | awk '{print $1}'):8000/v1/models
```
### test on your local machine

#### find out the hostname on the remote gpu instance

```bash
hostname
```

```bash
#example:
$ hostname
alvis4-41
```

#### setup tunnel on your local machine
```bash
ssh -N -L <local_port>:<hostname_of_gpu_instance>:<remote_port>  <hostname_of_remote>
```

```bash
#example:
$ ssh -N -L 8000:alvis4-41:8000  alvis2
```

### test from your local machine
```bash
curl http://localhost:8000/v1/models
```

```bash
#example:
$ curl http://localhost:8000/v1/models
{"object":"list","data":[{"id":"./models/Qwen3-32B-FP8","object":"model","created":1751877546,"owned_by":"vllm","root":"./models/Qwen3-32B-FP8","parent":null,"max_model_len":40960,"permission":[{"id":"modelperm-afe147129dea4bffb7d5cdc88052b790","object":"model_permission","created":1751877546,"allow_create_engine":false,"allow_sampling":true,"allow_logprobs":true,"allow_search_indices":false,"allow_view":true,"allow_fine_tuning":false,"organization":"*","group":null,"is_blocking":false}]}]}% 
```

## Developing 

Alvis provide a number of different ways of developing on the system. To keep sane I recommend you put the blindfolds on and only work with apptainer.
