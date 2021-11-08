---
title: 'How to Manage Workflow with Resource Constraint on HPC'
author: Sichong Peng
date: '2021-11-08'
slug: "how-to-manage-workflow-with-resource-constraint"
excerpt_separator: <!--more-->
categories:
  - Workflow
tags:
  - Workflow
  - HPC
---

> Note: This is an update to my previous post: [How to Run Snakemake pipeline on HPC]({% post_url 2019-10-17-how-to-run-snakemake-pipeline-on-hpc %}).

In my [previous post]({% post_url 2019-10-17-how-to-run-snakemake-pipeline-on-hpc %}), I disucessed some tips on how to effectively manage workflow using Snakemake on an HPC system. However, I have recently noticed that Snakemake support for `--cluster-config` is offcially deprecated in favor of `--profile`. I spent most of today digging into this feature and now I'm happy to share with you my latest setup.
<!--more-->
# How does `--profile` differ from `--cluster-config`?
In short, `--profile` is more universal and versatile. You can set different profiles for different environments: HPC, local machines, cloud computing, etc. 

# How to make a simple Slurm profile

To use a profile, you should first create a directory:
```{bash}
mkdir -p ~/.config/snakemake/slurm
```
Here I name my profile as `slurm` indicating that it's a profile for executing on HPC with Slurm.

Then in the newly created folder, make a new file `config.yaml`:
```
jobs: 10
cluster: sbatch
use-conda: true
```

Then when you run snakemake with: `snakemake --profile slurm`, Snakemake will search your current directory as well as `~/.config/snakemake/` for a directory named `slurm` and attempt to load `config.yaml` inside that directory. Once it is able to load the file, it will execute with above options specified.

Now as you may have guessed, `profiles` effectively allows you to put command line arguments for Snakemake in a file that can be re-used at any time and from any directory. 

Now to port the workflow we have from last post ([check it out here](/2019/10/17/how-to-run-snakemake-pipeline-on-hpc/)), we will make our Slurm profile's `config.yaml` look like this:
```{yaml}
jobs: 10
cluster: "sbatch -p high -t {resources.time_min} --mem={resources.mem_mb} -c {resources.cpus} -o logs_slurm/{rule}_{wildcards} -e logs_slurm/{rule}_{wildcards} --mail-type=FAIL --mail-user=user@mail.com"
```

Then in our `Snakefile`, we will modify rules like this:
```
samples = ['sample1', 'sample2', 'sample3', 'sample4']
rule all:
    input: expand("Alignment/{sample}.bam", sample = samples)

rule downloadFASTQ:
    output: "RawData/{sample}_1.fastq.gz", "RawData/{sample}_2.fastq.gz"
    resources: time_min=60, mem_mb=100, cpus=1
    shell:
     """
     ascp username@host:path/to/file/{wildcards.sample} RawData/
     """

rule mapFASTQ:
    input: 
        f1 = "RawData/{sample}_1.fastq.gz", 
        f2 ="RawData/{sample}_2.fastq.gz", 
        ref = "Ref/ref.fa"
    output: temp("Alignment/{sample}.sam")
    resources: time_min=300, mem_mb=8000, cpus=8
    shell:
     """
     bwa mem -@ {resources.cpus} -R "@RG\\tID:{wildcards.sample}\\tSM:{wildcards.sample}" {input.ref} {input.f1} {input.f2} > {output}
     """

rule convertSAM:
    input: "Alignment/{sample}.sam"
    output: "Alignment/{sample}.bam"
    resources: time_min=60, mem_mb=2000, cpus=1
    shell:
     """
     samtools view -bh {input} > {output}
     """
```

Now we can simply run Snakemake with `snakemake --profile slurm`!

# Use default resource assignment
You probably noticed that in the above I used `resources` directive instead of `params` directive to define memory, cpus, and time each rule should request. This is because `resources` directive enables an awesome feature in Snakemake that allows you to make full use of an HPC system's mighty power (more on this later)!

But how can we set default resource allocation like we did before with `--cluster-config` so that we don't have to mannually set those numbers for every rule?

We can do so in our Slurm profile's `config.yaml`:
```{yaml}
jobs: 10
cluster: "sbatch -p high -t {resources.time_min} --mem={resources.mem_mb} -c {resources.cpus} -o logs_slurm/{rule}_{wildcards} -e logs_slurm/{rule}_{wildcards} --mail-type=FAIL --mail-user=user@mail.com"
default-resources: [cpus=1, mem_mb=2000, time_min=60]
```

And now in our `Snakefile` we only need to set `resource` when a rule requires an amount that is different from default:
```
samples = ['sample1', 'sample2', 'sample3', 'sample4']
rule all:
    input: expand("Alignment/{sample}.bam", sample = samples)

rule downloadFASTQ:
    output: "RawData/{sample}_1.fastq.gz", "RawData/{sample}_2.fastq.gz"
    resources: mem_mb=100
    shell:
     """
     ascp username@host:path/to/file/{wildcards.sample} RawData/
     """

rule mapFASTQ:
    input: 
        f1 = "RawData/{sample}_1.fastq.gz", 
        f2 ="RawData/{sample}_2.fastq.gz", 
        ref = "Ref/ref.fa"
    output: temp("Alignment/{sample}.sam")
    resources: time_min=300, mem_mb=8000, cpus=8
    shell:
     """
     bwa mem -@ {resources.cpus} -R "@RG\\tID:{wildcards.sample}\\tSM:{wildcards.sample}" {input.ref} {input.f1} {input.f2} > {output}
     """

rule convertSAM:
    input: "Alignment/{sample}.sam"
    output: "Alignment/{sample}.bam"
    resources: time_min=60, mem_mb=2000, cpus=1
    shell:
     """
     samtools view -bh {input} > {output}
     """
```

# Parallelize jobs based on resource consumption

HPC system usually has resource constraint for users in order to mantain general accessibility for all users. 

Let's say we have access to 30 CPUs and 500Gb RAM. And we wish to run a workflow with following steps:

1. Run FastQC on raw files
2. Run BWA on raw files


Based on past experiences, we can have some expectations for these steps:

1. FastQC cannot be multithreaded for a single input file and therefore does not benefit from having more than 1 CPU. But it runs relatively fast and requires only a small amount of RAM.
2. Depending on input file size and complexity, BWA can take a very long time and typically requires a substantially large RAM. BWA is multithreaded so having more CPUS can greatly decrease running time.

Say we use 1 CPU, 2Gb RAM for FastQC and 5 CPUS, 30Gb RAM for BWA and we simply use `-j` option from Snakemake to parallelize jobs, we either sprout too many jobs for BWA or too few for FastQC. While on an HPC system submitting more jobs than total allocated resources will simply result in jobs in queue, this can be a real problem with cloud computing or a local machine: jobs will be terminated by the OS if it runs out of RAM!

So how do we make most of resources available to us without crashing everything? Luckily, `resources` directive in Snakemake allows us a more versatile way of managing resource allocation on an HPC system. We can define resource consumption on a per-rule basis and an overall resource constraint at workflow level. With such information, Snakemake will then wait to submit any jobs if executing it will exceed any resource constraint.

Let's add resource constraint to our Slurm profile:
```{yaml}
jobs: 100
cluster: "sbatch -p high -t {resources.time_min} --mem={resources.mem_mb} -c {resources.cpus} -o logs_slurm/{rule}_{wildcards} -e logs_slurm/{rule}_{wildcards} --mail-type=FAIL --mail-user=user@mail.com"
default-resources: [cpus=1, mem_mb=2000, time_min=60]
resources: [cpus=30, mem_mb=500000]
```

And then our `Snakefile`:

```{yaml}
import pandas as pd
samples = pd.read_csv("...") # A list of 100 samples
rule FastQC:
    input: "Data/{sample}.fastq.gz"
    output: "Results/QC/{sample}_fastqc.zip"
    shell:
     """
     fastqc -o Results/QC/ {input} 
     """
     
rule BWA:
    input: "Data/{sample}.fastq.gz"
    output: "Results/mapping/{sample}.sam"
    resources: cpus=5, mem_mb=30000, time_min=1440
    shell:
     """
     bwa mem -t {resources.cpus} -o {output} /path/to/ref.fa {input}
     """
```

Now we can invoke Snakemake with our profile: `snakemake --profile slurm`. With these resource constraints, Snakemake will at most submit 6 BWA jobs (30/5=6) or 30 FastQC jobs (30/1=30). 

# Use different partitions for different rules
Many HPC systems with Slurm have a queue system with different partitions. Usually these will be assigned different priorities and one can specify which partition they wish their job to be queued on when submitting a job with `-p` argument for `sbatch` command. In our previous post, we used `high` for all our rules. A typical partition system with `high`, `med`, and `low` partitions may look like this:

- `high`: Jobs start as soon as there are sufficient resources. Can suspend jobs running on `med` or `low` partitions to free up resources. One usually have limited amount of resources available to them on this partition (eg. 30 CPUS, 500Gb RAM) and when one's total jobs exceed that limit, jobs will wait in queue.

- `med`: Jobs start as soon as there are sufficient resources available. Can suspend jobs running on `low` partition to free up resources. One may have same access limitation as `high` partition. When a job on `med` partition is suspended by a job on `high` partition, it can resume as soon as its resource is freed up.

- `low`: Jobs start as soon as there are sufficient resources available. Typically "free-for-all" access. When a job on `low` partition is suspended by a job on `med` or `high` partition, it is ["preempted"](https://slurm.schedmd.com/preempt.html) and will restart once its resources are freed up.

Low priority partitions are great for small jobs that finish fast and don't require large amount of resources. We can put our FastQC jobs on this partition so that it doesn't compete for resources with our BWA jobs! 

To do this, let's modify our Slurm profile again:

```{yaml}
jobs: 100
cluster: "sbatch -p {params.partition} -t {resources.time_min} --mem={resources.mem_mb} -c {resources.cpus} -o logs_slurm/{rule}_{wildcards} -e logs_slurm/{rule}_{wildcards} --mail-type=FAIL --mail-user=user@mail.com"
default-resources: [cpus_high=0, mem_mb_high=0, time_min=60, cpus=1, mem_mb=2000]
resources: [cpus_high=30, mem_mb_high=500000]
```

Notice here we use two sets of resources: `cpus` + `mem_mb` and `cpus_high` + `mem_mb_high`. 
Since we only want to count resources consumption towards resource limit when a job is run on a `high` or `med` priority, we separate them out and use `cpus` and `mem_mb` to submit jobs to Slurm while set `cpus_high` and `mem_mb_high` only for jobs on `high` and `med` partition so they get counted towards resource limit.

```{yaml}
import pandas as pd
samples = pd.read_csv("...") # A list of 100 samples
rule FastQC:
    input: "Data/{sample}.fastq.gz"
    output: "Results/QC/{sample}_fastqc.zip"
    params: partition = 'low'
    shell:
     """
     fastqc -o Results/QC/ {input} 
     """
     
rule BWA:
    input: "Data/{sample}.fastq.gz"
    output: "Results/mapping/{sample}.sam"
    params: partition = 'med'
    resources: cpus=5, mem_mb=30000, time_min=1440, cpus_high=5, mem_mb_high=30000
    shell:
     """
     bwa mem -t {resources.cpus} -o {output} /path/to/ref.fa {input}
     """
```

Now when we invoke Snakemake with `snakemake --profile slurm`, all FastQC jobs will be submitted at once while BWA jobs will be parallelized to up to 5 at a time.

# Set default values for partition

If we have a workflow with dozens or even more rules, is there a way to set default partitions for all my rules so I don't have to define them for every single rule in my workflow? Sadly, at the moment, there is no official support for such a feature in Snakemake. However, several workarounds do exist.

Fisrt, we can use an input function for `params` directive to determine partition based on a separate configuration file. For example, we may have a `cluster_partition.conf`:
```
__default__: low
BWA: med
```

And then in our `Snakefile`:
```
import pandas as pd
import yaml
samples = pd.read_csv("...") # A list of 100 samples
with open('cluster_partition.conf', 'r') as f:
    partitions = yaml.safe_load()
    
def get_partition(wildcards, rule_name):
    return partitions[rule_name] if rule_name in partitions else partitions['__default__']

rule FastQC:
    input: "Data/{sample}.fastq.gz"
    output: "Results/QC/{sample}_fastqc.zip"
    params: partition = get_partition('FastQC')
    shell:
     """
     fastqc -o Results/QC/ {input} 
     """
     
rule BWA:
    input: "Data/{sample}.fastq.gz"
    output: "Results/mapping/{sample}.sam"
    params: partition = get_partition('BWA')
    resources: cpus=5, mem_mb=30000, time_min=1440, cpus_high=5, mem_mb_high=30000
    shell:
     """
     bwa mem -t {resources.cpus} -o {output} /path/to/ref.fa {input}
```

Alternatively, we can use a custom wrapper to submit our job scripts. [Here is an excellent example](https://github.com/Snakemake-Profiles/slurm) showcasing the potentials of Snakemake.