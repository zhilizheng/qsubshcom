# qsubshcom

An easy job submitter for PBS Pro (IMB), Torque (Tinaroo, Flashlite), SGE (old QBI) and Slurm(new QBI); it works on all clusters based on these system, not for UQ only.

qsubshcom will determine the cluster type automatically, and call the correct submit command. 

It can run across all of these cluster, you don't have to rewrite the script again and again in order to run on different cluster.
     
Author: Zhili

License: MIT License (See README.md and LICENSE)

Note: the original qsubshcom script referenced a little portion of code from QBI IT. However, the call routines are rewrote totally. You shall pay attention to the parameter changes, and much more robust enhancement on the job dependency and cluster type dedution.

If you find some bugs, you can create an issue here. I will fix it if I have time. 

### update:

Jan 26, 2020: Add Slurm support.

Apr 12, 2018: Add openMP env variable, that can read from program which support. GCTA 1.91.4 support this feature, it would run in multiple thread from the number of cores specified in this script even without the --thread-num.

Nov 28, 2017: Add support of QSUBSHCOM\_EXTRAS variables. You can put cluster dependant variable here, such as put export QSUBSHCOM\_EXTRAS="-acct=UQ-IMB-CNSG" into your ~/.bashrc. These variables will append into the submission task in qsubshcom automatically. Note: QSUBSHCOM\_EXTRAS will overwrite the 6th parameters in qsubshcom, not put too much variables here.  

Nov 27, 2017: Change the manner to determine the grid engine, as some cluster has multiple grid engine; remove the default account (UQ-IMB-\*\*\*), we should add -acct=ACCOUNT\_NAME to specify, as it is different in each cluster; -ntype can't work properly, it may have been obsoleted.

Sep 21, 2017: Add TRI cluster support

Sep 18, 2017:  update ${TASK\_ID} that can be find in the script run by qsubshcom "bash your\_scrip.sh" or qsubshcom "sh your\_script.sh"...

## usage
Download the qsubshcom, and put it into your $PATH, e.g. $HOME/bin, chmod 700 qsubshcom

qsubshcom command["one command |; two command"] num\_CPU[1] total\_memory[2G] task\_name wait\_time[1:00:00] other\_params

If you lots of old codes already, you can run it: qsubshcom "bash Your\_script.sh" 2 4G you\_task\_name 10:00:00 ""    .  qsubshcom will submit the command in Your\_script.sh to the computing node with 2 CPU core, 4GB memory (in total, not per CPU core), and a walltime of 10 hours.  The PBS or SGE commands in Your\_script.sh will be ignored (such as ${PBS\_ARRAY\_INDEX}), so you'd better remove the cluster dependent variables, it will be replaced by empty unexpectly!!! If you run the job array in your script, you can relace the ${PBS\_ARRY\_INDEX} to ${TASK\_ID}, it will works on all cluster. 

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
It will be rounded up into larger one in Torque and SGE if MEM/num\_CPU is not integer, such as 
```
qsubshcom "echo hello" 3 5G test 00:00:05 "" 
```
It will allocate 6G memory in total (not 6G each CPU!)

Note: the memory limitation here is virtual memory enforced by some cluster engine.                                            

Just use 1G, 10M without B, some cluster does not support GB or MB.                                                                       
### other_params:                                                                                                                                                                                
* -wait=JOB1:JOB2:JOB3 wait these jobs finished. qsubshcom will determine these JOB ID exist or not.                                                                                                                                             
* -array=1-100:2   create a job array that run task 1 3 5 7 ... 99                                                                                                                             
* -log=log\_name, log the job submission into log file. default is qsub\_time.log                                                                                                                
* -ntype=node\_type, specify the node type into node_type (Torque only, Tinaroo: -ntype=Special to use group special queue). Note: this flag can't work in current cluster setup.                                                                                                                         
* -queue=queue\_type. Specify the queue type                                                                                                                                                    
* -acct=ACCOUNT\_NAME. Specify the account, equal to "#PBS -A ACCOUNT_NAME"

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
