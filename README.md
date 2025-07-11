# Working with Alvis (and other NAISS affiliated supercomputers) for researchers at the Karolinska Institute.

Basically, how to get started doing inference and developing agents.

Hopefully, helpful info for budding AI researchers at KI.

This is for doing interactive development. These instructions are not for you if you need long-running jobs or already have an established use case. 

## Get Access

Apply at <https://supr.naiss.se/>.

You must, in general, be a PhD student or above. Ask your supervisor to apply on your behalf if you ain't qualified, yet ;).

## Getting started

### For super beginners

* If you haven't used the terminal before – when, in the instructions you encounter `<user_id>` it means that the whole thing including the arrows needs to be replaced. 
* Watch some youtube videos about how to work faster with the terminal, or what the best terminal setup is.
* Remember that you always can ask your favorite chat-llm for help explaining the different commands.

## Local SSH Config

### Test that your credentials work

```
ssh <user_id>@alvis2.c3se.chalmers.se
```

```
# example
# On the local machine:
$ ssh user_id@alvis2.c3se.chalmers.se
user_id@alvis2.c3se.chalmers.se's password: 

# On the remote machine:
   █  █▙   █ ▟  █  █      alvis2 is a shared login node
   █  █▜▙  █▟▛  █  █  ▟   Use with care and submit jobs
   █  █ ▜  █▛▟  █  █ ▟█   to the queue.
  ▟█  █    █▟▛  █  █▟▛█
 ▟▛█  █    █▛   █  █▛ █
 ▛ █  █    █    █  ▛  █
   █  █    █    █     █

For support, see https://www.c3se.chalmers.se/support

For intro, see https://indico.chalmers.se/category/39/
```

### Set up an SSH key

You can follow instructions at [GitHub](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent) on how to generate an ssh key.

Then add the public key to ~/.ssh/authorized_keys on Alvis.

Try to avoid using old RSA signatures and `scp` as in the [C3se example as of 2025-07-07](https://www.c3se.chalmers.se/documentation/connecting/ssh/#setting-up-a-hostname-alias) and instead use ed25519 and `ssh-copy-id`. More secure and faster.

```bash
# example
# On the local machine:
$ ssh-keygen -t ed25519 -a 100 -C "your_email@domain"
$ ssh-copy-id -i ~/.ssh/id_ed25519.pub user_id@alvis2.c3se.chalmers.se
```

I don't think it's needed to update the permissions manually if you use ssh-copy-id instead of scp. But if needed (mening you still are forced to use your password) one can do. 

```bash
# example
# On the remote machine:
$ ls -ld ~/.ssh ~/.ssh/authorized_keys
drwx------ 2 user_id user_id 2880 Jul  7 12:02 /cephyr/users/user_id/Alvis/.ssh
-rw------- 1 user_id user_id   90 Jul  7 12:02 /cephyr/users/user_id/Alvis/.ssh/authorized_keys

## if not showing drwx------ for ~/.ssh and -rw------- for ~/.ssh/authorized_keys then do
$ chmod 700 ~/.ssh
$ chmod 600 ~/.ssh/authorized_keys
```

### add an Alvis login node to your ~/.ssh/config 

```bash
# example
# On the local machine:
mkdir ~/.ssh # if not existing already (but it should, after you've run ssh-keygen)
cd ~/.ssh
touch config # if not existing already
vi config # or edit with some other editor you prefer
```

and add

```bash
Host alvis2
    IdentityFile   ~/.ssh/id_ed25519
    IdentitiesOnly yes           # only use the key(s) you list    
    HostName alvis2.c3se.chalmers.se
    User <replace_with_your_username>
    ControlMaster auto
    ControlPersist yes
    RequestTTY yes 
    ControlPath ~/.ssh/ssh_mux_%L_%h_%r
```

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
$ sinfo <as above...>
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

## Get a node with one (to four) GPU(s)

TODO: add a multi-node example

### Get your project id

First, get your project id. You can use `projinfo` to find it if you don't already know it.

```bash
projinfo
```

```bash
# example
# On the remote machine:
$ projinfo
 Project                Used[h]         Allocated[h]      Queue
    User
---------------------------------------------------------------
NAISS1234-prj-id          18.97                  250      alvis
    my_user               18.97   
```

### Get an interactive allocation

Allocate yourself a GPU node.

```bash
salloc -A <REPLACE_WITH_YOUR_PROJECT_ID> -N1 --gres=gpu:A100:1 --time=3:00:00 
```

```bash
# example
# On the remote machine
$ salloc -A NAISS_proj_id -N1 --gres=gpu:A100:1 --time=3:00:00 
salloc: Granted job allocation 4765049
salloc: Waiting for resource configuration
salloc: Nodes alvis4-39 are ready for job
```
If a node of the specified type isn't available, you will be automatically queued.

You can modify the `:1` part after `--gres=gpu:A100:1` to request for more GPUs on that node. I.e `--gres=gpu:A100:4` for four.

You can also change the `:A100` part if you want a node with a different type of GPU.

See the [docs](https://slurm.schedmd.com/salloc.html) for more nice arguments.

### shell into the allocated node 

(don't ssh into the node as it can mess with the slurm scheduler if you have not yet tied up the allocation)

```bash
srun --jobid=<job_id> --overlap --pty bash -l 
```

```bash
# example
# On the remote machine
srun --jobid=4765049 --overlap --pty bash -l 
```

#### find your jobid

If you somehow lost track of your jobid (it's printed when you salloc) you can find all your running jobs.

```bash
squeue -u $USER -O jobid,state,nodelist
```

```bash
# example
# On the remote machine
$ squeue -u $USER -O jobid,state,nodelist
JOBID               STATE               NODELIST            
4765049             COMPLETING          alvis3-35           
```


### check that you have the right allocation
```bash
# example
# On the remote machine, 
nvidia-smi
```

```bash
# example
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

# 0MiB/81920MiB indicates that we have an a100fat allocated and ready to use
```

## Run inference
### download/setup your dev environment

TODO: add an example of downloading models and containers

### cd to container location

Alvis has a storage solution called Mimer that is available on both login nodes and the GPU nodes. It's very convenient to store everything related to your project in that space.

```bash
cd /mimer/NOBACKUP/groups/<project_id>
```

### start container

```bash
apptainer shell --nv --bind <path_to_models_on_host>:<path_to_models_in_container> <name_and_path_to_appcontainer>
```

```bash
# example
# On the remote machine, in the GPU instance:
$ apptainer shell --nv --bind ./models:/models vllm-openai_latest.sif
```

### start vllm with tool use and tool auto support

This is an example for running on 1 GPU. There are many options for VLLM. See the docs or ask you fav llm for suggestions.

```bash
# example
# On the remote machine, inside the container:
$ vllm serve ./models/Qwen3-32B-FP8  --enable-auto-tool-choice --tool-call-parser hermes --reasoning-parser deepseek_r1 --host 0.0.0.0 --port 8000
```

* `--enable-auto-tool-choice` is a good default as some agentic framworks require it. If you use a framework that do requrie it then you would get a 400 bad request when calling without it.
* modify the `--tool-call-parser` if you are using another model that support another parser than `hermes`. Most models have there own parser for this.
* remove `--enable-auto-tool-choice` and `--tool-call-parser hermes` if you don't want tool calling or your model don't support it
* remove `--reasoning-parser deepseek_r1` if you don't want reasoning
* modify the `--reasoning-parser` if you are using another model that support another parser than `deepseek_r1`.

### after "INFO: Application startup complete." you can test by curling the HTTP server from the host of the container

But you can also skip this and test from your local machine right away – see below.

``` bash
curl http://$(hostname -I | awk '{print $1}'):8000/v1/models
```

### test on your local machine

#### find out the hostname on the remote gpu instance

```bash
hostname
```

```bash
# example
# On the remote machine:
$ hostname
alvis4-41
```

#### Set up a tunnel on your local machine
```bash
ssh -N -L <local_port>:<hostname_of_gpu_instance>:<remote_port>  <hostname_of_remote>
```

```bash
# example
# On the local machine:
$ ssh -N -L 8000:alvis4-41:8000  alvis2
```

#### test from your local machine
```bash
curl http://localhost:8000/v1/models
```

```bash
# example
# On the local machine:
$ curl http://localhost:8000/v1/models
{"object":"list","data":[{"id":"./models/Qwen3-32B-FP8","object":"model","created":1751877546,"owned_by":"vllm","root":"./models/Qwen3-32B-FP8","parent":null,"max_model_len":40960,"permission":[{"id":"modelperm-afe147129dea4bffb7d5cdc88052b790","object":"model_permission","created":1751877546,"allow_create_engine":false,"allow_sampling":true,"allow_logprobs":true,"allow_search_indices":false,"allow_view":true,"allow_fine_tuning":false,"organization":"*","group":null,"is_blocking":false}]}]}% 
```

## Congrats!

You should now be able to connect to your model on your local machine as if it were on your own localhost. Happy promting.

## Stop GPU Allocation

You can recover any GPU hours left if you are done before the requested time has run out. This of course a good practice.

First find the jobid.

```bash
squeue -u $USER -O jobid,state,nodelist
```

Then cancel it.

```bash
scancel <jobid>
```

## Caveats

### These instructions

I wrote these instructions after the fact, so some of the setup is from memory and not tested. Feel free to reach out if you encounter any problems or have suggestions.

### Working remotely (VPN)

When SSH-ing to Alvis you need to be on Sunet. You are automatically on Sunet on the KI wired network or eduroam. 

However, when working from home you need a suitable VPN. Sadly, the KI VPN won't help you out. What you can do instead is to use the Chalmers credentials provided to you when getting access to Alvis to connect through the Chalmers VPN (eduVPN).

### GPUs on Alvis

Alvis, sadly, doesn't have any Hopper or Blackwell GPUs so try to get access to Berzelius if you can. Performance and setup will be a lot better/easier on GPUs of a newer generation than Ampere.

### Stick to Bash

Alvis is happy to provide zsh and fish but unless you know what you are doing stick to bash. The interaction between shells, slurm, containers, etc. is annoying to keep track of and fix.

### ulimit

vLLM will complain about ulimit – me knowing this limit can't be raised without bribing the admins. Though, hopefully you should be fine with the default limit.

### Sensitive data

At KI we deal with a lot of sensitive data (remember, even pseudonymized data is sensitive data). A NAISS-affiliated supercomputer might not be a good choice for that kind of data. Make sure to consider this.

