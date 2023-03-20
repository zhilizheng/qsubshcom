# qsubshcom

An easy job submitter for PBS Pro (IMB), Torque (Tinaroo, Flashlite), SGE (old QBI) and Slurm(new QBI); it works across all clusters based on these system, not for UQ only. 

qsubshcom will determine the cluster type automatically, and call the correct submit command. You don't have to rewrite the script again and again for different clusters.

Author: Zhili

License: MIT License (See README.md and LICENSE)

If you find some bugs, you can create an issue here. I will fix it if I have time. 

### Update:
Dec 3,  2020: Update the default SGE PE, SGE PE shall be changed if the admin had a different setting

Feb 16, 2020: Fixed a bug when choosing a cluster engine.

Jan 26, 2020: Added Slurm support.

Apr 12, 2018: Exposed the number of cores to openMP OMP_NUM_THREADS variable. GCTA from 1.91.4 supports this feature.

Nov 28, 2017: Added the support of QSUBSHCOM\_EXTRAS variables. You can put cluster dependant variables here, such as put export QSUBSHCOM\_EXTRAS="-acct=UQ-IMB-CNSG" into your ~/.bashrc (e.g. for RCC clusters). These variables will be appended to qsubshcom command automatically. Note: QSUBSHCOM\_EXTRAS will overwrite the 6th parameters in qsubshcom, do not put too many variables here.  

Nov 27, 2017: Changed the manner to determine the grid engine, as some cluster has multiple grid engine; remove the default account (UQ-IMB-\*\*\*), we should add -acct=ACCOUNT\_NAME to specify, as this is different in each cluster; -ntype can't work properly, it is obsoleted.

Sep 21, 2017: Added TRI cluster support

Sep 18, 2017: Passed ${TASK\_ID} to the scripts that run by qsubshcom "bash your\_scrip.sh" or qsubshcom "sh your\_script.sh"...

## Usage
Download the qsubshcom, and put it into your $PATH, e.g. $HOME/bin, chmod 700 qsubshcom

qsubshcom command["one command |; two command"] num\_CPU[1] total\_memory[2G] task\_name wait\_time[1:00:00] other\_params

qsubshcom will go to the folder where you run the qsubshcom command, thus the cd WORK_DIR is not necessary. It will return the job id, and log the commands and job id to qsub_TIME.log in your working folder. All job running information from the cluster goes to job_reports folder, named by jobName\_time...

If you have lots of existing codes, you can run by: qsubshcom "bash Your\_script.sh" 2 4G you\_task\_name 10:00:00 ""     qsubshcom will submit the command in Your\_script.sh to a node with 2 CPU core, 4GB memory (in total, not per CPU core), and a walltime of 10 hours.  The PBS or SGE commands in Your\_script.sh will be ignored (such as `${PBS_ARRAY_INDEX}`), so cluster dependent variables should be removed, it may be replaced by empty string unexpectly on some other cluster system!!! If you run the job array in your script, you can relace the `${PBS_ARRY_INDEX}` to `${TASK_ID}`, it will work across supported clusters. 

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

### Command
We'd like to use "" to surround the command. 

We can specify the command directly:  qsubshcom "plink --bfile ..." ...

Or save the command in a temp variable:
```{bash}
temp_command="gcta64 --bfile test --make-grm --out test_grm"
qsubshcom "$temp_command" 1 1G test_grm 10:00:00 ""
```

If you would like to pass some special character into the job scripts directly, they shall be escaped, or it would be interpretd on the head node, not the working nodes. such as $  " . Store the variable in a temp variable first, both $ and other special vars shall use just single \ to escape;

```{bash}
temp_command="awk '{print \$1, \$2} test.txt > test2.txt"
qsubshcom "$temp_command" 1 1G test_awk 00:00:05 ""
```
### Memory
The memory requirements will be rounded up into larger one in Torque or SGE if MEM divided by num\_CPU is not an integer, such as 
```
qsubshcom "echo hello" 3 5G test 00:00:05 "" 
```
It will allocate 6G memory in total in Torque or SGE (not 6G each CPU!), but 5G in the other cluster system.

Note: the memory limitation here is virtual memory enforced by some cluster engine.                                            
Just use 1G, 10M without B, some cluster does not accept GB or MB.                                                                       

### Other_params:

* -wait=JOB1:JOB2:JOB3 wait these jobs to finish, and then run. qsubshcom will determine these JOB ID exist or not.                                                                                                                                         
* -array=1-100:2   create a job array that run task 1 3 5 7 ... 99                                                                                                                             
* -log=log\_name, log the job submission into log file. default is qsub\_time.log                                                                                                                
* -ntype=node\_type, specify the node type to node_type (Torque only, Tinaroo: -ntype=Special to use group special queue). Note: this flag is obsoleted.                                                                                                                         
* -queue=queue\_type. Specify the running queue.                                                                                                                                                  
* -acct=ACCOUNT\_NAME. Specify the account, equivalent to "#PBS -A ACCOUNT\_NAME".

* -ge=CLUSTER\_ENGINE. Specify the type of cluster engine manually, could be PBS(for PBSpro), SGE(Sun grid engine), TOR(Torque), SLM (Slurm). We don't need to specify this flag usually, it will be determined by your cluster automatically.

* You can put any other cluster engine supported parameters here, it will pass directly into the qsub command. e.g. -l host=host1:host2                                                        
Note: these parameters specified by yourself (such as -l host) may not be supported in other cluster engine. We shall avoid using this type of parameters if not necessary.

example: "-wait=123:124:125 -array=1-2 -l host=host1:host2"                                                                                                                   
 
## Examples
```
# run awk, note: $ shall be escaped by \, 
qsubshcom "awk -v OFS='\t' '{print \$1,\$2}' <<< 'test test2' > {TASK_ID}" " 1 1G test2 10:00:00 "-array=1-2"

# run a bash script into the cluster
qsubshcom "bash ./test.sh" 1 1G test_script 00:05:00 ""
```

Here is an exmaple to run GWAS with qsubshcom
```
geno=A
# GRM
grm_pid=`qsubshcom "gcta64 --bfile $geno --chr {TASK_ID} --make-grm --out geno_grm_chr{TASK_ID} --thread-num 5" 5 10G grm 5:00:00 "-array=1-22"` 

for i in $(seq 1 22); do
    echo "geno_grm_chr$i" >> mgrm.txt
done
# need to wait until GRM is done. 
merge_GRM=`qsubshcom "gcta64 --mgrm mgrm.txt --make-grm --out geno_grm" 1 10G merge_grm 1:00:00 "-wait=$grm_pid"`

# need to wait until merge GRM is done.
pca_pid=`qsubshcom "gcta64 --grm geno_grm --pca 20 --out geno_pca --thread-num 5" 5 20G pca 5:00:00 "-wait=$merge_GRM"`

# assciation
assoc_command="plink --bfile $geno --logistic --covar geno_pca --out geno_assoc --threads 5"
# We don't need to job ID any more, just run it. 
qsubshcom "$assoc_command" 5 20G assoc 10:00:00 "-wait=$pca_pid"
