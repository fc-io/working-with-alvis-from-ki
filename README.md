# Working with Alvis for researchers at the Karolinska Institute.

Hopefully, helpful info for AI researchers at KI.

How to get started developing agents and doing inference on the Alvis supercomputer.

This is for doing variable interactive development. If you need long running jobs, or have a set use case then use more scripts and sbatch.

## Get Access

Apply for acess at https://supr.naiss.se/

You need to be at least at the level of phd (or get help from your supervior) to be granted an allocation.

## Caveats

### Working remotly (VPN)

When ssh-ing to alvis you need to be on sunet. You are automatically on sunet on the KI wired network or eduroam. 

However, when working from home you need a sutable VPN. Sadly the KI VPN won't help you out. What you can do instead is to use the Chalmers credentials provided to you when getting access to Alvis to connect through chalmers VPN (eduVPN).

## Gettings started

## local ssh config

TODO: add alvis to ~/.ssh/config
TODO: setup credentials

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

First, get your project id. You can use `projinfo` to find it.

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

## shell into the allocated node (don't ssh into the node as it can mess with the slurm scheduler if you have not yet tied up the allocation)
```
srun --pty bash -l 
```
# if you want to connect from another node change the jobid to the 
## first, quick check, to see all running jobs
## then use srun to connect to your instance/job
```
$ squeue -u $USER -O jobid,state,nodelist
```

example:

```
squeue -u $USER -O jobid,state,nodelist
JOBID               STATE               NODELIST            
4732902             COMPLETING          alvis3-35           
```

```
srun --jobid=[jobid] --overlap --pty bash -l 
```

# check that you have the right allocation
```
nvidia-smi
```
```
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

## download/setup your dev environment

TODO: add an example

## cd to container location

cd /mimer/NOBACKUP/groups/<project_id>

# start container
apptainer shell --nv --bind ./models:/models vllm-openai_latest.sif
vllm serve ./models/Qwen3-32B-FP8  --reasoning-parser deepseek_r1 --host 0.0.0.0 --port 8000

## after "INFO:     Application startup complete." test by curling the http server from the host of the container (alvis4-41 for example)
``` bash
curl http://$(hostname -I | awk '{print $1}'):8000/v1/models
```
# find out the hostname
```
$ hostname
> alvis4-41
```
## setup tunnel on local machine
```
ssh -N -L 8000:alvis4-41:8000  alvis2
```

## test
curl http://localhost:8000/v1/models

vllm serve ./models/Qwen3-32B-FP8 --reasoning-parser deepseek_r1

## Devloping 

Alvis provide a number of different ways of developing on the system. Most of them are subpar. All of them really, but it could be worse. To keep sane I recommend you put the blindfolds on and only work with apptainer.
