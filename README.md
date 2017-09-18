# qsubshcom

The job submitter for PBS Pro (IMB), Torque (Tinaroo, Flashlite) and SGE (QBI). 

qsubshcom will determine the cluster type automatically, and call the correct qsub command. 

It can run across all of these cluster, you don't have to rewrite the script again and again in order to run on different cluster.
     
Author: Zhili

License: MIT License (See README.md and LICENSE)

Note: the original qsubshcom script referenced a little portion of code from QBI IT. However, the call routines are rewritted totally. You shall pay attention to the parameter changes, and much more robust enhancement on the job dependency and cluster type dedution.

If you find some bugs, you can create an issue here. I will fix it if I have time. 

## usage
Download the qsubshcom, and put it into your $PATH, e.g. $HOME/bin, chmod 700 qsubshcom

qsubshcom command["one command |; two command"] num_CPU[1] total_memory[2G] task_name wait_time[1:00:00] other_params

If you lots of old codes already, you can run it: qsubshcom "bash Your_script.sh" 2 4G you_task_name 10:00:00 ""    .  qsubshcom will submit the command in Your_script.sh to the computing node with 2 CPU core, 4GB memory (in total, not per CPU core), and a walltime of 10 hours.  The PBS or SGE commands in Your_script.sh will be ignored (such as ${PBS_ARRAY_INDEX}), so you'd better remove the cluster dependent variables, it will be replaced by empty unexpectly!!! If you run the job array in your script, you can relace the ${PBS_ARRY_INDEX} to ${TASK_ID}, it will works on all cluster. 

example:
```{bash}
# This will echo hello world from 1 to 5 into test1.txt test2.txt ... in 5 jobs, each job with one cpu, 1GB memory and wall-time 5 seconds
qsubshcom "echo \"hello world {TASK_ID}\" >> test{TASK_ID}.txt" 1 1G test_hello 00:00:05 "-array=1-5"

command="echo hello {TASK_ID}"
# run this command, but save the qstat submitted ID into hellp_pid
hello_pid=`qsubshcom "$command" 1 1G test_hello2 00:00:05 "-array=1-5"`
# run test_hello3, wait until test_hello2 finished
qsubshcom "echo hello3" 1 1G test_hello3 00:00:05 "-wait=$hello_pid"

# the job running logs are in ./job_reports
# the submit log are in ./qsub_TIME.log
```

### update:
Sep 18, 2017:  update ${TASK_ID} that can be find in the script run by qsubshcom "bash your_scrip.sh"

### command
We'd like to use "" to surround the command. 

We can specify the command directly:  qsubshcom "plink --bfile ..." ...

Or save the command in a temp variable:
```{bash}
temp_command="gcta64 --bfile test --make-grm --out test_grm"
qsubshcom "$temp_command" 1 1G test_grm 10:00:00 ""
```

If you would like to pass some special character into the job scripts directly, they shall be escaped, otherwise it will interpret on the head node, not the working nodes. such as $  " . Store the variable in a temp variable first, both $ and other special vars shall use just single \ to escape;

```{bash}
temp_command="awk '{print \$1, \$2} test.txt > test2.txt"
qsubshcom "$temp_command" 1 1G test_awk 00:00:05 ""
```
### memory
It will be rounded up into larger one in Torque and SGE if MEM/num_CPU is not integer, such as 
```
qsubshcom "echo hello" 3 5G test 00:00:05 "" 
```
It will allocate 6G memory in total (not 6G each CPU!)

Note: the memory limitation here is virtual memory enforced by some cluster engine.                                            

Just use 1G, 10M without B, some cluster does not support GB or MB.                                                                       
### other_params:                                                                                                                                                                                
* -wait=JOB1:JOB2:JOB3 wait these jobs finished. qsubshcom will determine these JOB ID exist or not.                                                                                                                                             
* -array=1-100:2   create a job array that run task 1 3 5 7 ... 99                                                                                                                             
* -log=log_name, log the job submission into log file. default is qsub_time.log                                                                                                                
* -ntype=node_type, specify the node type into node_type (Torque only, Tinaroo: -ntype=Special to use group special queue)                                                                                                                         
* -queue=queue_type. Specify the queue type                                                                                                                                                    
* You can put any other cluster engine supported parameters here, it will pass directly into the qsub command. e.g. -l host=host1:host2                                                        
Note: these parameters specified by yourself (such as -l host) may not be supported in other cluster engine. We shall avoid using this types of parameters

example: "-wait=123:124:125 -array=1-2 -ntype=Special -l host=host1:host2"                                                                                                                   
 
## Examples
```
# run awk, note: $ shall be escaped by \, 
qsubshcom "awk -v OFS='\t' '{print \$1,\$2}' <<< 'test test2' > {TASK_ID}" " 1 1G test2 10:00:00 "-array=1-2"

# run a bash script into the cluster
qsubshcom "bash ./test.sh" 1 1G test_script 00:05:00 ""
```

Here is a exmaple to run GWAS with qsubshcom
```
geno=A
# GRM
grm_pid=`qsubshcom "gcta64 --bfile $geno --chr {TASK_ID} --make-grm --out geno_grm_chr{TASK_ID} --thread-num 5" 5 10G grm 5:00:00 "-array=1-22"` 
# merge GRM. merge_pid need further rewrite to get rid of the .grm.bin and .grm.N.bin. (pipe with sed unique)
merge_pid=`qsubshcom "ls -1 geno_grm_chr* > mgrm.txt" 1 1G list 00:00:05 "-wait=$grm_pid"`
merge_GRM=`qsubshcom "gcta64 --mgrm mgrm.txt --make-grm --out geno_grm" 1 10G merge_grm 1:00:00 "-wait=$merge_pid"`

pca_pid=`qsubshcom "gcta64 --grm geno_grm --pca 20 --out geno_pca --thread-num 5" 5 20G pca 5:00:00 "-wait=$merge_GRM"`

# assciation
assoc_command="plink --bfile $geno --logistic --covar geno_pca --out geno_assoc --threads 5"
# Because we need not chain it, don't need the qsub job id, it can run by this
qsubshcom "$assoc_command" 5 20G assoc 10:00:00 "-wait=$pca_pid"
