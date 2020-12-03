#!/bin/bash
#===============================================================================
# Job submitter for PBS Pro, Toruqe, SGE
# Author Zhili:  zhilizheng@outlook.com
# License: MIT License (See README.MD)

# Easy job submitter for PBS, Torque, SGE and Slurm
# usage:   qsubshcom command["one command |; two command"] num_CPU[1] memory[2G] task_name wait_time[1:00:00] other_params
# command: {TASK_ID} for job array index, we'd like to use "" to surround the command
# if call qsubshcom directly, then the $ should be escaped with \$ ;
# if call it in `` then $ should have double \\$, other special vars are not affected,just single \
# if we store the variable in a temp variable first, just single \ to escape

# memory will auto roll up into larger one in Torque and SGE, the memory limitation here is virtual memory enforced by some cluster engine.
# Just use 1G, 10M without B, some cluster does not support GB or MB.

# other_params:
# -wait=JOB1:JOB2:JOB3 wait these jobs finished
# -array=1-100:2   create a job array that run task 1 3 5 7 ... 99
# -log=log_name, log the job submission into log file. default is qsub_time.log
# -ntype=node_type, specify the node type into node_type (Torque only), currently not work
# -acct=account_name, specify the account name
# -queue=queue type. Specify the queue type
# You can put any other cluster engine supported parameters here, it will pass directly into the qsub command. e.g. -l host=host1:host2
# example: "-wait=123:124:125 -array=1-2 -l host=host1:host2"

# qsubshcom "awk -v OFS='\t' '{print \$1,\$2}' <<< 'test test2' > {TASK_ID} |; echo -e \"\${TASK_ID}\nhello\" " 1 1G test2 10:00:00 "-array=1-2"
#===============================================================================
scriptname=$(mktemp)
if [[ $# != 6 ]]; then
    echo "Error when submitting the job, params: $@"
    exit 1
fi
if [ -z $4 ]; then
        echo "No job name"
        exit
else
        name=$4
fi

mkdir -p ./job_reports
today=`date +%Y.%m.%d_%H.%M.%S`
todayLOG=`date +%Y.%m.%d`

additions="$6 ${QSUBSHCOM_EXTRAS}"

GE=`sed -nr 's/^.*-ge=(\S*)\s?.*$/\1/p' <<< $additions`
if [ -z "$GE" ]; then
    pbs_file=`command -v qorder`
    if [[ -n "$pbs_file" ]]; then
        pbs_folder=`dirname $pbs_file`
        GE="PBS"
        if [[ -f "$pbs_folder/qchkpt" ]]; then
            GE="TOR"
        fi
    fi

    ge_file=`command -v qsh`
    if [[ -n "$ge_file" ]]; then
        GE="SGE"
    fi

    slm_file=`command -v sbatch`
    if [[ -n "$slm_file" ]]; then
	 GE="SLM"
    fi
fi

if [ -z "$GE" ]; then
    echo "Sorry, we can't recognize the grid engine"
    echo "You can use -ge=TOR|SGE|PBS|SLM in the 6th parameter to specify"
    exit 1
fi

mem_raw="$3"
mem=`printf "%s\n" "${mem_raw//[![:digit:]]/}"`
mem_unit=`printf "%s\n" "${mem_raw//[[:digit:]]/}"`
if [ -z "$mem_unit" ]; then
    echo "Sorry, you have to specify the memory unit (3 parameter) in GB or MB or KB"
    exit 1
fi
if [ "$GE" == "TOR" ] || [ "$GE" == "SLM" ]; then
    MEM_UNIT="${mem_unit^^}"
    if [ "${MEM_UNIT: -1}" != "B" ]; then
        MEM_UNIT="${MEM_UNIT}B"
    fi
    mem_unit="$MEM_UNIT"
fi

mem_core=`bc <<< "($mem + $2 - 1)/$2"`

ntype=`sed -nr 's/^.*-ntype=(\S*)\s?.*$/\1/p' <<< $additions`
if [[ -n "$ntype" ]]; then
    ntype=":NodeType=$ntype"
fi

queue=`sed -nr 's/^.*-queue=(\S*)\s?.*$/\1/p' <<< $additions`

account=`sed -nr 's/^.*-acct=(\S*)\s?.*$/\1/p' <<< $additions`

echo '#!/bin/bash' > $scriptname
if [ "$GE" == "SGE" ]; then
    echo "#$ -N ${name}" >> $scriptname
    echo "#$ -S /bin/bash" >> $scriptname
    echo "#$ -j y" >> $scriptname
    echo "#$ -o ./job_reports/" >> $scriptname
    echo "#$ -cwd" >> $scriptname
    echo "#$ -V" >> $scriptname
    echo "#$ -pe thread $2" >> $scriptname
    echo "#$ -l h_vmem=${mem_core}${mem_unit}" >> $scriptname
    echo 'export TASK_ID=${SGE_TASK_ID}' >> $scriptname
    #echo "#$ -l time=$5" >> $scriptname
fi

if [ "$GE" == "PBS" ] || [ "$GE" == "TOR" ]; then
    echo "#PBS -N ${name}" >> $scriptname
    echo "#PBS -S /bin/bash" >> $scriptname
    #echo "#PBS -j oe" >> $scriptname
    echo "#PBS -V" >> $scriptname
    echo "#PBS -l select=1:ncpus=$2${ntype}:mem=${mem}${mem_unit}" >> $scriptname
    if [ -n "$5" ];then
        echo "#PBS -l walltime=$5" >> $scriptname
    fi
    if [[ -n "$queue" ]]; then
        echo "#PBS -q $queue" >> $scriptname
    fi
    if [[ -n "$account" ]]; then
        echo "#PBS -A $account" >> $scriptname
    fi
fi

if [ "$GE" == "SLM" ]; then
    echo "#SBATCH --job-name=${name}" >> $scriptname
    echo "#SBATCH --ntasks=1" >> $scriptname
    echo "#SBATCH --cpus-per-task=$2" >> $scriptname
    echo "#SBATCH --mem=${mem}${mem_unit}" >> $scriptname
    echo "#SBATCH --time=$5" >> $scriptname
    if [[ -n "$queue" ]]; then
        echo "#SBATCH --partition $queue" >> $scriptname
    fi	
    if [[ -n "$account" ]]; then
	echo "#SBATCH -A $account" >> $scriptname
    fi
fi


if [ "$GE" == "PBS" ]; then
    if [[ "$additions" =~ ^.*-array.*$ ]]; then
        echo "#PBS -o ./job_reports/${name}_${today}_A^array_index^.log" >> $scriptname
        echo "#PBS -e ./job_reports/${name}_${today}_A^array_index^.err" >> $scriptname
    else
        echo "#PBS -o ./job_reports/${name}_${today}.log" >> $scriptname
        echo "#PBS -e ./job_reports/${name}_${today}.err" >> $scriptname
    fi
    #echo "#PBS -l nodes=1:ppn=$2${ntype},mem=${mem}${mem_unit}" >> $scriptname
fi

if [ "$GE" == "TOR" ]; then
    echo "#PBS -o ./job_reports/${name}_${today}.log" >> $scriptname
    echo "#PBS -e ./job_reports/${name}_${today}.err" >> $scriptname
    #echo "#PBS -l nodes=1:ppn=$2${ntype},pmem=$mem_core$mem_unit" >> $scriptname
fi

if [ "$GE" == "SLM" ]; then
    if [[ "$additions" =~ ^.*-array.*$ ]]; then
        echo "#SBATCH -o ./job_reports/${name}_${today}_A%a.log" >> $scriptname
        echo "#SBATCH -e ./job_reports/${name}_${today}_A%a.err" >> $scriptname
    else
        echo "#SBATCH -o ./job_reports/${name}_${today}.log" >> $scriptname
        echo "#SBATCH -e ./job_reports/${name}_${today}.err" >> $scriptname
    fi
fi
	

if [ "$GE" == "PBS" ] || [ "$GE" == "TOR" ]; then
    echo 'cd $PBS_O_WORKDIR' >> $scriptname
    echo 'export TASK_ID=${PBS_ARRAY_INDEX}' >> $scriptname
fi

if [ "$GE" == "SLM" ]; then
    echo 'export TASK_ID=${SLURM_ARRAY_TASK_ID}' >> $scriptname
fi

echo -e "cat << 'EOF' \n PARAM to here: $@ \nEOF" >> $scriptname
echo 'echo ""' >> $scriptname
echo 'echo ========QSUB STD out========' >> $scriptname

echo "export OMP_NUM_THREADS=$2" >> $scriptname
sed "s/\$\?{TASK_ID}/\${TASK_ID}/g" <<< $1 | sed "s/\s*|;\s*/\n/g" >> $scriptname 
echo 'echo =========QSUB STD end=========' >> $scriptname
echo 'echo ""' >> $scriptname
echo 'echo ""' >> $scriptname
echo 'echo ""' >> $scriptname
echo 'echo ""' >> $scriptname
echo 'echo ""' >> $scriptname

params=$scriptname
if [ -n $"$additions" ]; then
    raw_param=$additions
    wait_jb=`sed -nr 's/^.*-wait=(\S*)\s?.*$/\1/p' <<< $raw_param`
    arrays=`sed -nr 's/^.*-array=(\S*)\s?.*$/\1/p' <<< $raw_param`
    log=`sed -nr 's/^.*-log=(\S*)\s?.*$/\1/p' <<< $raw_param`
    remain_param=`echo $raw_param | sed 's/-ge=\S*//g' | sed 's/-log=\S*//g' | sed 's/-wait=\S*//g' | sed 's/-array=\S*//g' | sed 's/-ntype=\S*//g' | sed 's/-queue=\S*//g' | sed 's/-acct=\S*//g'`
    if [ "$GE" == "PBS" ];then
        if [[ -n "$arrays" ]];then
            params="-J $arrays $params";
        fi
        if [[ -n "$wait_jb" ]];then
            wait_list=""
            IFS=':' read -ra jobs <<< "$wait_jb"
            for job in "${jobs[@]}";do
                if [[ -n `qstat -f $job` ]];then
                    wait_list="$wait_list:$job"
                fi
            done
            if [[ -n "$wait_list" ]];then
                params="-W depend=afterok$wait_list $params"
            fi
        fi
    elif [ "$GE" == "SGE" ];then
        if [[ -n "$arrays" ]];then
            params="-t $arrays $params"
        fi
        if [[ -n "$wait_jb" ]];then
            wait_jb=`sed 's/:/,/g' <<< "$wait_jb"`
            params="-hold_jid $wait_jb $params"
        fi
    elif [ "$GE" == "TOR" ];then
        if [[ -n "$arrays" ]];then
            params="-t $arrays $params"
        fi
        if [[ -n "$wait_jb" ]];then
            wait_list=""
            IFS=':' read -ra jobs <<< "$wait_jb"
            for job in "${jobs[@]}";do
                if [[ -n `qstat -f $job` ]];then
                    if [[ $job == *"[]"* ]];then
                        job_id=`qsubshcom "echo stupid" 1 10MB qwait_ar 00:00:30 "-W depend=afterokarray:$job $remain_param"`
                        wait_list="$wait_list:$job_id"
                    else
                        wait_list="$wait_list:$job"
                    fi
                fi
            done
            if [[ -n "$wait_list" ]];then
                params="-W depend=afterok$wait_list $params"
            fi
        fi
    elif [ "$GE" == "SLM" ]; then
	if [[ -n "$arrays" ]]; then
	    params="--array=$arrays $params"
	fi
        if [[ -n "$wait_jb" ]]; then
            wait_list=""
            IFS=':' read -ra jobs <<< "$wait_jb"
            for job in "${jobs[@]}";do
                if squeue -j $job | grep -q $job;then
                    wait_list="$wait_list:$job"
                fi
            done
            if [[ -n "$wait_list" ]]; then
                params="--dependency=afterok$wait_list $params"
            fi
        fi
            
    fi
    params="$remain_param $params"
fi
echo 'echo ................Trans params..........' >> $scriptname
echo -e "cat << 'EOF' \n $params \nEOF" >> $scriptname
echo 'echo ..........STD error begin.....' >> $scriptname
if [ "$GE" == "SLM" ]; then
    output=`sbatch $params`
else
    output=`qsub $params`
fi

if [ -z "$log" ]; then
    log="qsub.$todayLOG.log"
else
    log="$log.$todayLOG.log"
fi

echo "$name" >> $log
echo "QSUB_TIME: $today" >> $log
echo "QSUB_OUT: $output" >> $log
printf "QSUB_JOB_ID: " >> $log

if [ "$GE" == "SGE" ]; then
    echo $output | sed -r 's/Your (job|job-array) ([0-9]*).*/\2/'
    echo $output | sed -r 's/Your (job|job-array) ([0-9]*).*/\2/' >> $log
fi

if [ "$GE" == "PBS" ]; then
    echo $output
    echo $output >> $log
fi

if [ "$GE" == "TOR" ]; then
   echo $output | tail -n 1 | sed 's/\n//'
   echo $output | tail -n 1 >> $log
fi

if [ "$GE" == "SLM" ]; then
    echo $output | sed -r 's/.* ([0-9]*).*/\1/g'
    echo $output | sed -r 's/.* ([0-9]*).*/\1/g' >> $log
fi

echo "QSUB_PARAMS: $@" >> $log
echo "QSUB_SUBMIT: $params" >> $log
echo "-----------------------------" >> $log
echo " " >> $log

rm $scriptname
