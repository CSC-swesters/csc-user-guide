# Array jobs

In many cases, a computational analysis job contains a number of similar independent subtasks. A user may have several datasets that are analyzed in the same way, or the same simulation code is executed with a number of different parameters. This kind of tasks are often called as _embarrassingly parallel_ jobs as the task can in principle be distributed to as many processors as there are subtasks to run. In Puhti, these tasks can be effectively run by using the array job function of the SLURM batch job system.

## Defining an array job

In SLURM, an array job is defined using the option `--array` or `-a`, e.g.
```
#SBATCH --array=1-100
```
will launch not just one batch job, but 100 batch jobs where the subjob specific environment variable __$SLURM_ARRAY_TASK_ID__ has a value ranging from 1 to 100. This variable can then be utilized in the actual job launching commands so that each subtask gets processed. All subjobs are launched in the batch job system at once, and they are executed using as many processors as are available.

In addition to defining a job range, you can also provide a list of job index values, e.g.
```
#SBATCH --array=4,7,22
```
would launch three jobs with __$SLURM_ARRAY_TASK_ID__ values 4, 7 and 22.

You can also include step size in the job range definition. The array job definition
```
#SBATCH --array=1-100:20
```
would run five jobs with __$SLURM_ARRAY_TASK_ID__ values: 1, 21, 41, 61 and 81.

In some cases, it may be reasonable to limit the number of simultaneously running processes. This is done with the notation `%max_number_of_jobs`. For example, in a case were you have 100 jobs but a license for only five simultaneous processes, you could ensure that you will not run out of license using the definition
```
#SBATCH --array=1-100%5
```
 
## A simple array job example

As a first array job example, let's assume that we have 50 datasets (data_1.inp, data_2.inp … data_50.inp) that we would like to analyze using the program _my_prog_ that uses the syntax
```
my_prog inputfile outputfile
```
Each of the subtasks requires less than two hours of computing time and less than 4 GB of memory. We can perform all 50 analysis tasks with the following batch job script:
```
#!/bin/bash -l
#SBATCH --job-name=array_job
#SBATCH --output=array_job_out_%A_%a.txt
#SBATCH --error=array_job_err_%A_%a.txt
#SBATCH --account=<project>
#SBATCH --partition=small
#SBATCH --time=02:00:00
#SBATCH --ntasks=1
#SBATCH --mem-per-cpu=4000
#SBATCH --array=1-50

# run the analysis command
my_prog data_${SLURM_ARRAY_TASK_ID}.inp data_${SLURM_ARRAY_TASK_ID}.out
```
In the batch job script, the line `#SBATCH --array=1-50` defines that 50 subjobs will be submitted. The other #SBATCH lines refer to individual subjobs. In this case, one subjob uses at most one processor (`--ntasks=1`), 4 GB of memory (`--mem-per-cpu=4000`), and can last up to two hours (`--time=02:00:00`). However, the total wall clock time needed to process all 50 tasks is not limited.

In the job execution commands, the script utilizes the __$SLURM_ARRAY_TASK_ID__ variable in the definition of the input and output files so that the first subjob will run the command
```
my_prog data_1.inp data_1.out
```
the second one will run the command
```
my_prog data_2.inp data_2.out
```
and so forth.

The job can be now launched using the command
```
sbatch job_script.sh
```
Typically, not all jobs are executed at once. However, after a while, a large number of jobs may be running simultaneously. When the batch job is finished, the _data_dir_ directory contains 50 output files.

After submitting an array job, the command
```
squeue -l -u <username>
```
reveals that you have one pending job and possibly several other jobs running in the batch job system. All these jobs have a _jobid_ that contains two parts: the _jobid_ number of the array job and the subjob number. Directing the output of each subjob into a separate file is recommended as the file system may fail if several dozens of processes try to write into the same file at the same time. If the output files need to be merged into one file, it can often be easily done after the array job has finished. For example in the case above, we could collect the results into one file using the command
```
cat data_*.out > all_data.out
```
In the case of standard output and error files, defined in the _SBATCH_ lines,  you can use the definitions `%A` and `%a` to give unique names to the output files of each subjob. In the file names, _%A_ will be replaced by the ID of the array job and _%a_ will be replaced by the __$SLURM_ARRAY_TASK_ID__.

## Using a file name list in an array job

In the example above, we were able to use __$SLURM_ARRAY_TASK_ID__ to refer to the order numbers in the input files. If this type of approach is not possible, a list of files or commands created before the submission of the batch job can be used. Let's assume that we have a similar task as defined above but the file names do not contain numbers but are in the format _data_aa.inp_, _data_ab.inp_, _data_ac.inp_ and so forth. Now, we first need to make a list of the files to be analyzed. In this case, we could collect the file names into the file _namelist_ using the command
```
ls data_*.inp > namelist
```
The command
```
sed –n <row_number>p inputfile
```
reads a certain line from the name list file. In this case, the actual command script could be
```
#!/bin/bash -l
#SBATCH --job-name=array_job
#SBATCH --output=array_job_out_%A_%a.txt
#SBATCH --error=array_job_err_%A_%a.txt
#SBATCH --account=<project>
#SBATCH --partition=small
#SBATCH --time=02:00:00
#SBATCH --ntasks=1
#SBATCH --mem-per-cpu=4000
#SBATCH --array=1-50

# set the input file to process
name=$(sed -n ${SLURM_ARRAY_TASK_ID}p namelist)
# run the analysis
my_prog ${name} ${name}.out
```
This example is otherwise similar to the first one except that it reads the name of the file to analyze from the file _namelist_. This value is stored in the variable _$name_, which is used in the job execution command. As the row number to read is defined by the __$SLURM_ARRAY_TASK_ID__, each data file listed in the file _namelist_ is processed as a different subjob. Note that as we now also use _$name_ in the output definition, the output file name will be in the format _data_aa.inp.out, data_ab.inp.out, data_ac.inp.out_ and so forth.

 
## Using array jobs in workflows with sbatch_commandlist
 
In Puhti, you can use the command `sbatch_commandlist` to execute a list of commands as an array job. This command takes as an input a text file. The command list is split into several pieces that are executed as an independent sub-task of a array batch job, automatically generated by the `sbatch_commandlist`. Thus one sub task of an array job may process several commands from the command list.

The syntax of this command is:
```text
sbatch_commandlist -commands commandlist
```

Options `-t` and `-mem` can be used to modify the time and memory reservation of the subjobs (default 12 h, 8GB).
By default the billing project is set based on the name of the scratch directory where this command is executed, but if needed, it can be assigned using option `-project`. The command list is split to in maximum 200 sub tasks. If individual tasks are very short you could use option `--max_jobs` to decrease the splitting so that each array job task would take at least around half an hour to process.

After submitting an array job, `sbatch_commandlist` starts monitoring the progress of the job. If you use `sbatch_commandlist` interactively in the login nodes, you normally don't want to keep the monitor running for hours. In these cases, you can just close the monitoring process by pressing `Ctrl-c`. The actual array job is not deleted, but it stays active in the batch job system and you can manage it with normal Slurm commands.

In addition to interactive usage, `sbatch_commandlist` can be utilized in batch jobs and automatic workflows, where only certain steps of the workflow can utilize array jobs based parallel computing. As an example, lets assume we have a gzip compressed tar-archive file my_data.tgz containing a directory with a large number of files. To create a new compressed archive, that includes also a md5 checksum file for each file we would need to: (1) un-compress and un-pack my_data.tgz,  (2) execute `md5sum` for each file and finally (3) pack and compress the my_data directory again. The second step of the workflow could be executed using a for-loop, but we could also use the loop just to generate a list of `md5sum` commands, that can be processed with `sbatch_commandist` .
```text
#!/bin/bash -l
#SBATCH --job-name=workfow
#SBATCH --output=workflow_out_%j.txt
#SBATCH --error=workflow_err_%j.txt
#SBATCH --account=<project>
#SBATCH --time=12:00:00
#SBATCH --mem=4000
#SBATCH --ntasks=1
#SBATCH --partition=small

#open the tgz file
tar zxf my_data.tgz
cd my_data

#generate a list of md5sum commands
for my_file in *
do
  echo "md5sum $my_file > $my_file.md5" >> md5commands.txt
done

#execute the md5commands as an array job
sbatch_commandlist -commands md5commands.txt

#remove the command file and compress the directory
rm -f md5commands.txt
cd ..
tar zcf my_data_with_md5.tgz my_data
rm -rf my_data
```
Note that the batch job script above is not an array job, but it launches a another batch job that is an array job.
