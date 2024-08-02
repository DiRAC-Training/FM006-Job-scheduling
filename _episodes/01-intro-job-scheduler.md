---
title: "Introduction to HPC Job Scheduling"
slug: "dirac-job-scheduling-intro-job-scheduling"
teaching: 0
exercises: 0
questions:
- "What is a job scheduler and why does a cluster need one?"
- "How do I find out what parameters to use for my Slurm job?"
- "How do I submit a Slurm job?"
- "What are DiRAC project allocations and how do they work?"
objectives:
- "Describe briefly what a job scheduler does."
- "Recap the fundamentals of Slurm job submission and monitoring."
- "Use Slurm and SAFE to determine the submission parameters to use to submit a job."
- "Summarise the main ways researchers may request time on a DiRAC facility."
keypoints:
- "A job scheduler ensures jobs are given the resources they need, and manages when and where jobs will run on an HPC resource."
- "Obtain your DiRAC account details to use for job submission from DiRAC's SAFE website."
- "Use `sinfo -s` to see information on all queues on a Slurm system."
- "Use `scontrol show partition` to see more detail on particular queues."
- "Use `sbatch` and `squeue` to submit jobs and query their status."
- "Access to DiRAC's resources is managed through the STFC's independent Resource Allocation Committee (RAC)."
- "Facility time may be requested through a number of mechanisms, namely in response to a Call for Proposals, the Director's Discretionary Award, and Seedcorn Time."
- "The [DiRAC website](https://dirac.ac.uk/getting-access/) has further information on the methods to request access, as well as application forms."
---


## job scheduling: A Recap

An HPC system might have thousands of nodes and thousands of users. How do we
decide who gets what and when? How do we ensure that a task is run with the
resources it needs? This job is handled by a special piece of software called
the _scheduler_. On an HPC system, the scheduler manages which jobs run where
and when.

The following illustration compares these tasks of a job scheduler to a waiter
in a restaurant. If you can relate to an instance where you had to wait for a
while in a queue to get in to a popular restaurant, then you may now understand
why sometimes your job does not start instantly as on your laptop.

![Compare a job scheduler to a waiter in a restaurant]({{ site.url }}{{ site.baseurl }}/fig/restaurant_queue_manager.svg){: width="650px"}

The scheduler used here is Slurm, and although
Slurm is not used everywhere, running jobs is quite similar
regardless of what software is being used. The exact syntax might change, but
the concepts remain the same.

### Fundamentals of Job Submission and Monitoring: A Recap

You may recall from a previous course, or your own experience, two basic Slurm commands;
namely `sbatch` to submit a job to an HPC resource, and `squeue` to query the state of a submitted job.
When we submit a job, we typically write a Slurm script which embodies the commands we wish to run on a compute node.
A job script (e.g. `basic-script.sh`) is typically written using the
[Bash shell](https://www.gnu.org/software/bash/manual/bash.html) language and a very minimal example looks something like this (save this in a file called `basic-script.sh`, but don't try to submit it just yet!):

~~~
#!/usr/bin/env bash
#SBATCH --account=yourAccount
#SBATCH --partition=aPartition
#SBATCH --nodes=1
#SBATCH --time=00:00:30

date
~~~
{: .language-bash}

The `#SBATCH` lines are special comments that provide additional information about our job to Slurm;
for example the account and partition, the maximum time we expect the job to take when running,
and the number of nodes we'd like to request (in this case, just one).
We'll look at other parameters in more detail later,
but let's focus on specifying a correct set of minimal parameters first.

#### `--account`

It's important to note that what you specify for the `--account` parameter is not your
machine login username or SAFE login; it's the *project account* to which you have access.
Project accounts are assigned an allocation of resources (such as CPU or disk space),
and to use them, you specify the project account code in the `--account` parameter.

You can find the projects to which you have access in your DiRAC's [SAFE account](https://safe.epcc.ed.ac.uk/dirac/).
To see them, after you login to SAFE, select `Projects` from the top navigation bar
and select one of your projects to see further details, where you'll find the project's
account code you can use.

#### `--partition`

The underlying mechanism which enables the scheduling of jobs across HPC systems like Slurm is that of the **queue** (or **partition**).
A queue represents a list of (to some extent) ordered jobs to be executed on HPC compute resources,
and sites often have many different queues which represent different aspects,
such as the level of prioritisation for jobs, or the capabilities of compute nodes
So when you submit a job, it will enter one of these queue to be executed.
How these queues are set up across multi-site HPC systems such as DiRAC can differ,
depending on local institutional infrastructure configurations, user needs, and site policies.

You can find out the queues available on a DiRAC site, and their current state, using:

~~~
sinfo -s
~~~
{: .language-bash}

The `-s` flag curtails the output to only a summary,
whereas omitting this flag provides a full listing of nodes in each queue and their current state.

You should see something like the following. This is an example from COSMA:

~~~
PARTITION         AVAIL  TIMELIMIT   NODES(A/I/O/T) NODELIST
cosma7               up 3-00:00:00     80/137/7/224 m[7229-7452]
cosma7-pauper        up 1-00:00:00     80/137/7/224 m[7229-7452]
cosma7-shm           up 3-00:00:00          0/1/0/1 mad02
cosma7-shm-pauper    up 1-00:00:00          0/1/0/1 mad02
cosma7-bench         up 1-00:00:00          0/0/1/1 m7452
cosma7-rp            up 3-00:00:00     82/140/2/224 m[7001-7224]
cosma7-rp-pauper*    up 1-00:00:00     82/140/2/224 m[7001-7224]
~~~
{: .output}

Here we can see the general availability of each of these queues (also known as partitions), as well as the maximum
time limit for jobs on each of these queues (in `days-hours:minutes:seconds` format).
For example, on the `cosma7` queue there is a 3-day limit, whilst the `cosma7-pauper` queue has a 1-day limit. The `*` beside the partition name indicates it is the default queue (although it's always
good practice to specify this explicitly).

> ## Queues on Other Sites
> 
> On other DiRAC sites, the queues displayed with `sinfo` will look different, for example:
>
> - Edinburgh's Tursa: there are `cpu` and `gpu` queues, as well as other gpu-specific queues.
> - Cambridge's CSD: there are a number of queues for different CPU and GPU architectures, such as `cclake`, `icelake`, `sapphire`, and `ampere`.
> - Leicester's DiAL3: there are two queues named `slurm` and `devel`, with `slurm` referring to the main HPC resource and `devel` for development or testing use (i.e. short running jobs).
{: .callout}

Note that the queues to which you have access will depend on your allocation setup, and this may not include the default queue (for example, if you only have access to GPU resources which are accessible on their own queue, like on Tursa, you'll need to use one of these queues instead).

To find out more information on queues, you can use the `scontrol show` command, which allows you to view the configuration of Slurm and it's current state.
So to see a breakdown of a particular queue, you can do (replacing `<partition_name>`):

~~~
scontrol show partition=<partition_name>
~~~
{: .language-bash}

So on COSMA, for example:

~~~
scontrol show partition=cosma7
~~~
{: .language-bash}

An example of the output, truncated for clarity:

~~~
PartitionName=cosma7
   AllowGroups=cosma7 AllowAccounts=do009,dp004,dp012,dp050,dp058,dp060,dp092,dp121,dp128,dp203,dp260,dp276,dp278,dp314,ds007 AllowQos=ALL
   AllocNodes=ALL Default=NO QoS=N/A
   DefaultTime=NONE DisableRootJobs=NO ExclusiveUser=NO GraceTime=0 Hidden=NO
   MaxNodes=UNLIMITED MaxTime=3-00:00:00 MinNodes=0 LLN=NO MaxCPUsPerNode=UNLIMITED
   Nodes=m[7229-7449]
   ...
~~~
{: .output}

In particular, we can see the accounts (under `AllowAccounts`) that have access to this queue
(which may display `ALL` depending on the queue and the system).

To see a complete breakdown of all queues you can use:

~~~
scontrol show partitions
~~~
{: .language-bash}

#### Other Parameters

Depending on your site and how your allocation is configured you may need to specify other parameters in your script.
On Tursa for example, you may need to specify a `--qos` parameter in the script which stands for quality of service, which is used to constrain or modify the characteristics of a job.
On other sites, a default `--qos` is already present and doesn't need to be explicitly supplied.

So for example on Tursa, we can use `scontrol show partition` to display the allowed QoS' for a particular queue, e.g.:

~~~
scontrol show partition=cpu
~~~
{: .language-bash}

~~~
PartitionName=cpu
   AllowGroups=ALL AllowAccounts=ALL AllowQos=sysadm,standard,high,short,debug,low,reservation
   AllocNodes=ALL Default=NO QoS=N/A
   DefaultTime=01:00:00 DisableRootJobs=NO ExclusiveUser=NO GraceTime=0 Hidden=NO
   MaxNodes=UNLIMITED MaxTime=2-00:00:00 MinNodes=0 LLN=NO MaxCPUsPerNode=UNLIMITED MaxCPUsPerSocket=UNLIMITED
   Nodes=tu-c0r0n[66-71]
   ...
~~~
{: .output}

We can see under `AllowQos` those that are permitted. So for example using the `cpu` queue on Tursa, in the batch script you may need to add a line containing `#SBATCH --qos=standard` (e.g. below the other `#SBATCH` directives) for jobs to work - make a note of this!

> ## Submitting a Job!
> 
> Let's check that we can first submit a job to Slurm so we can verify that we have a working set of
> Slurm parameters. As mentioned, these will vary depending on your circumstances, such as the projects
> to which you have access to on DiRAC, and the DiRAC site to which you are submitting jobs.
> 
> Once you've determined these, edit `basic-script.sh` (as shown above),
> substitute the correct `--account` and `--partition` values
> and add any additional parameters needed for your site (e.g. `--qos`), and then save the file.
> Next, submit that script using `sbatch basic-script.sh`.
> It may take some trial and error to find the correct parameters!
> Once you've successfully submitted the job, you should have a job identifier returned in the terminal
> (something like `309001`).
> 
> Lastly, you can use that job identifier to query the status
> of our job until it's completed using `squeue <jobid>` (e.g. `squeue 309001`).
> Once the job is complete, we are able to read the job's log file, typically held in a file named
> `slurm-<jobid>.out`, which show us any printed output from a job,
> and depending on the HPC system, perhaps other information regarding how and where the job ran.
{: .challenge}


## DiRAC Project Allocations

[Access to DiRAC's resources](https://dirac.ac.uk/getting-access/) is managed through the STFC's independent Resource Allocation Committee (RAC),
which provides access through allocations to researchers who request time on the facility.
There are a number of mechanisms through which facility time can be requested:

- **Call for full Proposals**: the RAC puts out an annual invitation to the UK theory and modelling communities to apply for computational resources on the DiRAC HPC Facility, with applications taking the form of scientific, technical, or Research Software Engineer support (RSE) time.
- **Director's Discretionary Award**: from time to time the DiRAC Director invites the UK theory and modelling communities to apply for discretionary allocations of computational resources. Discretionary time can also be applied for if you find you will be using your code at a larger scale than was previously requested in a full call for proposals. Applications can be made at any time.
- **Seedcorn Time**: for researchers who would like to get a feel for HPC, test and benchmark codes, or see what  DiRAC resources can do for you before making a full application, an application can be made for seedcorn time. Existing users may also apply for seedcorn allocations to enable code development/testing on a service that is not currently part of their project allocation. You can apply for Seedcorn Time at any time.

For more information regarding these options,
and for online application forms and contact details for enquiries, see the [DiRAC website](https://dirac.ac.uk/getting-access/).

Once the submission and its technical case has been approved,
allocations are managed within in 3-month chunks over the duration of the project,
which may be over a period of years.
Allocation usage is based primarily on core CPU hours or GPU hours.
Following a 3-month allocation project usage is reviewed and extended based on that review
(for instance, any non-use of the allocation during that 3-month window is queried,
with support provided to overcome any barriers to use.)
In addition, Research Software Engineering (RSE) support time may also be requested,
who can provide help with code optimisation, porting, re-factoring and performance analysis.
