---
title: Handle Edge Cases on HPC with Snakemake
author: Sichong Peng
layout: post
date: 
slug: 2021-11-12-handle-edge-cases-on-hpc-with-snakemake
categories:
  - Workflow
tags:
  - hpc
  - Workflow
---

Snakemake has great support for most HPC systems, via [custom submission command you can specify in profile]({% post_url 2021-11-08-how-to-manage-workflow-with-resource-constraint %}) and it works amazingly most of the time. However, occasionally it would fail to correctly recognize job status in some edge cases. 

For example, my HPC system implements preemption, which cancels jobs with lower priority when resources are required by higher-priority jobs and restarts them later on. This behaviour causes snakemake to regard the job as having failed. To handles cases like this, Snakemake provides a way to customize job status detection: `--cluster-status`

## How does `--cluster-status` script work?
This argument takes a python script that accepts job id and returns one of the three statuses: `success`, `running`, or `failed`.

For out preempted jobs, since they will be automatically rescheduled by Slurm (with same job id), we should tell Snakemake to treat those jobs as still running. This is especially important if you are using `--restart-times` in order to avoid duplicated job runs.

## An example cluster status script
Below is an example script that works well with my cluster configuration. You may need to change specific running statuses according to your cluster configuration.

```
#!/usr/bin/python
import sys, time, subprocess

jobid = sys.argv[1]

def check_state(output):
    running_status=["PENDING", "CONFIGURING", "COMPLETING", "RUNNING", "SUSPENDED", "PREEMPTED"]
    if "COMPLETED" in output:
        print("success")
    elif any(r in output for r in running_status):
        print("running")
    else:
        print("failed")

attempts = 5
delay = 3 # wait time (s) between attempts

for i in range(attempts):
    try:
        output = str(subprocess.check_output("sacct -j %s --format State --noheader | head -1 | awk '{print $1}'" % jobid, shell=True, universal_newlines = True).strip())
        if output:
            check_state(output)
            exit(0)
    except Exception as e:
        print("sacct error:" + e, file=sys.stderr)
    try:
        output = subprocess.run("scontrol show job -o " + str(jobid), capture_output = True, shell = True, universal_newlines = True)
        if output.stdout:
            info = {i.split('=')[0]: i.split('=')[1] for i in output.stdout.strip().split(' ')}
            check_state(info['JobState'])
            exit(0)
    except Exception as e:
        print(e, file=sys.stderr)
    time.sleep(delay)
if i >= attempts - 1:
    print('failed')
```

In this script I'm using two commands to prob job status: `sacct -j` and `scontrol show job -o`. This is because on my cluster, I often observe a breif delay (~10s) where `sacct -j` would return nothing when the job was just submitted. In that case, the script would fall back to `scontrol`. On the other hand, a job id only stays in `scontrol show job -o` for an unspecified amount of time (possibly undeterministic) after job completion. In that case, the script should fall back to `sacct`. 