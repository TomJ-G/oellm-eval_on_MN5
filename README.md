# Setting up oellm-eval on MN5
Since there is no internet access from MN5, I prepared the offline-installation package

Run command to enable uv:
`module purge && module load oneapi hdf5 python`

2. Copy `/gpfs/scratch/ehpc533/users/utu665365/oellm-eval-offline.tar.gz` or `/gpfs/scratch/ehpc533/users/utu665365/moe_build_offline.tar.gz` to your dir. oellm-eval-offline was build based on current main branch, and moe_build was based on Korbinian fork.
3.  Unpack `tar -xzvf oellm-eval-offline.tar.gz -C oellm-eval-offline`  and `cd /path/to/oellm-eval-offline`
4. Create venv `uv venv --python python3.12 .venv`
5. Install dependencies

```
uv pip install \
    --python .venv/bin/python \
    --no-index \
    --find-links ./wheelhouse \
    -r requirements.txt

uv pip install \
    --python .venv/bin/python \
    --no-index \
    --find-links ./wheelhouse \
    ./wheelhouse/oellm_eval-*.whl
```
    
6. Verify
```
.venv/bin/python -m pip check
.venv/bin/oellm-eval --help
```
7. ! Since current installation package was done before my update to clusters.yaml was proposed, you need to manually update the following: 
`/gpfs/scratch/ehpc533/users/$user/oellm-eval-offline/.venv/lib/python3.12/site-packages/oellm/resources/clusters.yaml` with either `clusters.yaml` from this repo, or manually:
```
marenostrum5:
  hostname_pattern:
    - "*alogin[0-9]*"
    - "*glogin[0-9]*"
    - "nid[0-9][0-9][0-9][0-9][0-9][0-9]"
  EVAL_BASE_DIR: "/gpfs/projects/ehpc533/oellm_shared_evals"
  PARTITION: "acc"
  QOS: "acc_ehpc"
  ACCOUNT: "ehpc533"
  QUEUE_LIMIT: 100 #debug: 100
  EVAL_CONTAINER_IMAGE: "MN5_v2.sif"
  SINGULARITY_ARGS: "--nv"
```
and also `/gpfs/scratch/ehpc533/users/$user/oellm-eval-offline/.venv/lib/python3.12/site-packages/oellm/resources/template.sbatch` either with `template.sbatch` from this repo or manually:

Update start of template with:
```
#!/bin/bash
#SBATCH --job-name=oellm-eval
#SBATCH --time={time_limit}
#SBATCH --gres=gpu:$GPUS_PER_NODE
#SBATCH --cpus-per-task=80
#SBATCH --output={log_dir}/%x-%A-%a.out
#SBATCH --partition=$PARTITION
#SBATCH --qos=$QOS
#SBATCH --account=$ACCOUNT
#SBATCH --error={log_dir}/%x-%A-%a.err
#SBATCH --array=0-{array_limit}%{max_array_len}
#SBATCH --nodes=$NODES
#SBATCH --nodelist=$NODELIST


unset PYTHONPATH
unset PYTHONHOME

module load singularity
# which singularity
singularity --version

CSV_PATH="{csv_path}"
NUM_JOBS={num_jobs}
TOTAL_EVALS={total_evals}
LIMIT="{limit}"
VENV_PATH="{venv_path}"
LM_EVAL_INCLUDE_PATH="{lm_eval_include_path}"
TASKS_PER_JOB={tasks_per_job}

# avoiding crashes due to compute nodes not having access to the internet
export HF_HOME=$HF_HOME
export HF_DATASETS_CACHE="$HF_HOME/datasets"
export HF_HUB_OFFLINE=1 #{hf_hub_offline}
export HF_HUB_ENABLE_HF_TRANSFER=0
```

and lastly `/gpfs/scratch/ehpc533/users/$user/oellm-eval-offline/.venv/lib/python3.12/site-packages/oellm/main.py`

Copy it from here or if you want to do changes manually:

Or update manually block around line 409, under comment `# Calculate dynamic array size and time limits` with:
```
    # Calculate dynamic array size and time limits
    total_evals = len(df)
    minutes_per_eval = 25  # Originally was 10 minutes per eval
    total_minutes = total_evals * minutes_per_eval
    max_minutes_per_job = 18 * 60  # 18 hours
    min_array_size_for_time = max(1, int(math.ceil(total_minutes / max_minutes_per_job)))
    desired_array_size = (
        min(max_array_len, total_evals) if total_evals >= max_array_len else total_evals
    )
    if desired_array_size < min_array_size_for_time:
        desired_array_size = min_array_size_for_time
    actual_array_size = min(remaining_queue_capacity, desired_array_size, total_evals)
    evals_per_job = max(1, int(math.ceil(total_evals / actual_array_size)))
    minutes_per_job = evals_per_job * minutes_per_eval
    minutes_with_margin = int(minutes_per_job * 1.5)
    hours_with_margin = max(1, int(math.ceil(minutes_with_margin / 60)))
    hours_with_margin = max(hours_with_margin, 4) # Originally this was 3
    hours_with_margin = min(hours_with_margin, 23)
    computed_time = f"{hours_with_margin:02d}:59:00" # Original was :59:00, changed for debug
    time_limit = computed_time
```
The above fragment is important as it controls how much time is scheduled per eval.

Container for running oellm-eval is already on MN5 shared location, so no need to copy anything

8. Before running eval execute (or add to your bashrc)
```
module purge && module load oneapi hdf5 python
unset PYTHONHOME
unset PYTHONPATH
export HF_HOME=/gpfs/projects/ehpc533/hf-cache
export HF_HUB_OFFLINE=1
```
The cache folder is shared, if there are any evaluation datasets missing you can upload them there
8. Test evaluation with 
`.venv/bin/oellm-eval schedule --models /gpfs/projects/ehpc533/annealing/models/baby_9b_dense_before-annealing/checkpoints/iter_0953312 --task_group "dclm-core-22"`

9. Shared evals folder is in `/gpfs/projects/ehpc533/oellm_shared_evals`
My results for dclm-core-22 and prelude_iter_0953312 are in `/gpfs/projects/ehpc533/oellm_shared_evals/utu665365/2026-11-18-13-07-38/results`
