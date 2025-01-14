# qsubshcom

A simple job submitter for PBS Pro, Torque, SGE and Slurm.

qsubshcom will find the cluster type automatically, and call the correct submit command without writting dialect (#PBS #SBATCH). It has a robust log system (e.g., all command you run, time), straightforward job dependency declaration (submit multiple job at same time and run some first), and real-time resource consumption monitor if not provided by your cluster. You don't have to rewrite the script again and again and struggling with job orders for different clusters.

qsubshcom has no dependent library in Linux system (if you have batch system installed). Download the qsubshcom, chmod +x qsubshcom, put it into your PATH, then it's ready to go. 

Author: Zhili
License: MIT, see LICENSE; no warrenty, no citaion needed.

## Installation
```{bash}
curl -O https://raw.githubusercontent.com/zhilizheng/qsubshcom/refs/heads/master/qsubshcom

chmod +x qsubshcom

# if $HOME/bin in your PATH
mv qsubshcom ~/bin/

qsubshcom
# prompt will be similar to below, it means installation success
## Error when submitting the job, params:
```

## Usage

```{bash}
# qsubshcom command["one command |; two command"] num_CPU[1] total_memory[2G] task_name run_time[1:00:00] other_params
# the job logs are in ./job_reports (*.log: stdout; *.err: stderr, *.mon: monitor information)
# job command was also logged in qsub_TIME.log
qsubshcom "echo 'hello world'" 1 1G helloTask 1:00:00 ""

# two or more commands at the same time
qsubshcom "echo 'hello world' ; echo 'hello2' |; echo 'hello3'" 1 1G helloTask 1:00:00 ""

#########################
## job dependency
# job1 and job2 run instantly
# job1 and job2 will save the job id
job1=`qsubshcom "echo 'hello world 1'" 1 1G helloTask1 1:00:00 ""`
job2=`qsubshcom "echo 'hello world 2'" 1 1G helloTask2 1:00:00 ""`

# job3 will start when job1 and job2 finished
command="echo hello world again"
job3=`qsubshcom "$command" 1 1G helloTask3 1:00:00 "-wait=$job1:$job2"`

# this job will start when job3 finished (also wait for job1 and job2)
# this is an array job, it will run 10 sub-jobs indexed from 1 to 10, {TASK_ID} is the array index
qsubshcom "echo 'hello world {TASK_ID}' > {TASK_ID}.log" 1 1G helloTask4 1:00:00 "-wait=$job3 -array=1-10"


#########################
## Run the script directly
# a sh script
qsubshcom "./test_script.sh" 1 1G demo 1:00:00 ""

# a Python script
qsubshcom "python3 test.py" 1 1G demoPy 1:00:00 ""

# a R script
qsubshcom "Rscript test.R --chr={TASK_ID}" 1 1G demoR 1:00:00 "-array=1-22"
```

If you have existing codes, you can run by: `qsubshcom "bash Your_script.sh" 2 4G you_task_name 10:00:00 ""`. qsubshcom will submit the command in Your\_script.sh to a node with 2 CPU core, 4GB memory (in total, not per CPU core), and a walltime of 10 hours.  The PBS or SGE commands in Your\_script.sh will be ignored (such as `${PBS_ARRAY_INDEX}`), so cluster dependent variables should be removed, it may be replaced by empty string unexpectly on some other cluster system!!! If you run the job array in your script, you can relace the `${PBS_ARRY_INDEX}` to `${TASK_ID}`, it will work across supported clusters. 

qsubshcom will go to the folder where you run the qsubshcom command, thus the `cd WORK_DIR` is not necessary. It will return the job id, and log the commands and job id to "qsub\_TIME.log" in your working folder. All job running information from the cluster goes to job\_reports folder, named by jobName\_timeXXX

## Example
```{bash}
chrs="1-22"
geno="test_chr{TASK_ID}"
pheno="pheno.txt"
snplist="chr{TASK_ID}.snplist"

genoQC1="extract_chr{TASK_ID}"
assocOut="test_assoc_chr{TASK_ID}"

# QC1 extract the SNPs out
cmd1="plink2 --bfile $geno --extract $snplist --make-bed --out $genoQC1"
pid1=`qsubshcom "$cmd1" 2 5G QC1 10:00:00 "-array=$chrs"`

# wait for the QC done
pid2=`qsubshcom "plink2 --bfile $genoQC1 --pheno $pheno --linear --out $assocOut" 2 5G assoc 10:00:00 "-array=$chrs -wait=$pid1"`

# processing the output
qsubhcom "Rscript test.R {TASK_ID}" 1 2G enrich 10:00:00 "-array=$chrs -wait=$pid2"

```

If you would like to pass some special character into the job scripts directly, they shall be escaped, or it would be interpretd on the head node, not the working nodes. such as $  " . Store the variable in a temp variable first, both $ and other special vars shall use just single \ to escape;

```{bash}
temp_command="awk '{print \$1, \$2} test.txt > test2.txt"
qsubshcom "$temp_command" 1 1G test_awk 00:00:05 ""
```

## Memory
The memory requirements will be rounded up into larger one in Torque or SGE if MEM divided by num\_CPU is not an integer, such as 
```
qsubshcom "echo hello" 3 5G test 00:00:05 "" 
```
It will allocate 6G memory in total in Torque or SGE (not 6G each CPU!), but 5G in the other cluster system.

Note: the memory limitation here is virtual memory enforced by some cluster engine.                                            
Just use 1G, 10M without B, some cluster does not accept GB or MB.                                                                       

## Other\_params:

* -wait=JOB1:JOB2:JOB3 wait these jobs to finish, and then run. qsubshcom will determine these JOB ID exist or not.                                                                                                                                         
* -array=1-100:2   create a job array that run task 1 3 5 7 ... 99                                                                                                                             
* -log=log\_name, log the job submission into log file. default is qsub\_time.log                                                                                                                
* -ntype=node\_type, specify the node type to node\_type (Torque only, Tinaroo: -ntype=Special to use group special queue). Note: this flag is obsoleted.                                                                                                                         
* -queue=queue\_type. Specify the running queue.                                                                                                                                                  
* -acct=ACCOUNT\_NAME. Specify the account, e.g., equivalent to "#PBS -A ACCOUNT\_NAME" or "#SBATCH -A"

* -ge=CLUSTER\_ENGINE. Specify the type of cluster engine manually, could be PBS(for PBSpro), SGE(Sun grid engine), TOR(Torque), SLM (Slurm). We don't need to specify this flag usually, it will be determined by your cluster automatically.

* You can put any other cluster engine supported parameters here, it will pass directly into the qsub command. e.g. -l host=host1:host2                                                        
Note: these parameters specified by yourself (such as -l host) may not be supported in other cluster engine. We shall avoid using this type of parameters if not necessary.

Example: "-wait=123:124:125 -array=1-2 -l host=host1:host2"                                                                                                                   

## Update:
Jan 14, 2025: Added the real-time job monitor, it will record the resource consumption every 10 seconds without any code, e.g., runtime, %CPU, physical memory (RSS), largest physical memory (PMEM)) in the background for all the tasks started by qsubshcom (logged in ./job\_reports/JOB\_NAME\_timeXXX.mon)

Dec 3,  2020: Updated the default SGE PE, SGE PE shall be changed if the admin had a different setting

Feb 16, 2020: Fixed a bug when choosing a cluster engine.

Jan 26, 2020: Added Slurm support.

Apr 12, 2018: Exposed the number of cores to openMP OMP\_NUM\_THREADS variable. GCTA from 1.91.4 supports this feature.

Nov 28, 2017: Added the support of QSUBSHCOM\_EXTRAS variables. You can put cluster dependant variables here, such as put export QSUBSHCOM\_EXTRAS="-acct=UQ-IMB-CNSG" into your ~/.bashrc (e.g. for RCC clusters). These variables will be appended to qsubshcom command automatically. Note: QSUBSHCOM\_EXTRAS will overwrite the 6th parameters in qsubshcom, do not put too many variables here.  

Nov 27, 2017: Changed the manner to determine the grid engine, as some cluster has multiple grid engine; remove the default account (UQ-IMB-\*\*\*), we should add -acct=ACCOUNT\_NAME to specify, as this is different in each cluster; -ntype can't work properly, it is obsoleted.

Sep 21, 2017: Added TRI cluster support

Sep 18, 2017: Passed ${TASK\_ID} to the scripts that run by qsubshcom "bash your\_scrip.sh" or qsubshcom "sh your\_script.sh"...

### Issues
Report bug by issues tab
