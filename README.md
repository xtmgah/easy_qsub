# easy_qsub

Easily submitting PBS job with script template, avoid repeatedly editing PBS scripts.

Inspired by [qtask](https://github.com/mbreese/qtask), support for **multiple inputs** 
is added (**See example 2), 3)**.). If "{}" appears in a command, it will be replaced
with the current filename. Four formats are supported.
For example, for a file named "/path/reads.fq.gz":

<table>
    <tr>
        <th>format</th>
        <th>target</th>
        <th>result</th>
    </tr>
    <tr>
        <td>{}</td>
        <td>full path</td>
        <td>/path/reads.fq.gz</td>
    </tr>
    <tr>
        <td>{%}</td>
        <td>basename</td>
        <td>reads.fq.gz</td>
    </tr>
    <tr>
        <td>{^.fq.gz}</td>
        <td>remove suffix from full path</td>
        <td>/path/reads</td>
    </tr>
    <tr>
        <td>{%^.fq.gz}</td>
        <td>remove suffix from basename</td>
        <td>reads</td>
    </tr>
</table>

**[Update]** It also support runing commands locally with option ```-lp``` (parallelly) or ```-ls``` (serially) 
for **multiple inputs**.

**[Update 2015-09-10]** To make best use of the support for multiple input, a script ```cluster_files``` is added to
cluster files into multiple directories by creating symbolic links or moving files.
**It's useful for programs which take one directory as input.**

Default template (```~/.easy_qsub/default.pbs```):

```
#PBS -S /bin/bash
#PBS -N $name
#PBS -q $queue
#PBS -l ncpus=$ncpus
#PBS -l mem=$mem
#PBS -l walltime=$walltime
#PBS -V

cd $$PBS_O_WORKDIR
echo run on node: $$HOSTNAME >&2
echo run: $cmd >&2

$cmd
```

Generated scripts are saved in ```/tmp/easy_qsub-user```.

## Installation

easy_qsub is a single script written in Python using standard library. 
It's Python 2/3 compatible, version 2.7 or later.

You can simply save the [script](https://raw.githubusercontent.com/shenwei356/easy_qsub/master/easy_qsub)
to directory included in environment PATH, e.g ```/usr/local/bin```.

Or
    
    git clone https://github.com/shenwei356/easy_qsub.git
    copy easy_qsub /usr/local/bin
    
## Usage

easy_qsub

```
usage: easy_qsub [-h] [-lp | -ls] [-N NAME] [-n NCPUS] [-m MEM] [-q QUEUE]                          
                 [-w WALLTIME] [-t TEMPLATE] [-o OUTFILE] [-v]                                      
                 command [files [files ...]]                                                        
                                                                                                    
Easily submitting PBS jobs with script template. Multiple input files                               
supported.                                                                                          
                                                                                                    
positional arguments:                                                                               
  command               command to submit                                                           
  files                 input files

optional arguments:
  -h, --help            show this help message and exit
  -lp, --local_p        run commands locally, parallelly
  -ls, --local_s        run commands locally, serially
  -N NAME, --name NAME  job name
  -n NCPUS, --ncpus NCPUS
                        cpu number [logical cpu number]
  -m MEM, --mem MEM     memory [2gb]
  -q QUEUE, --queue QUEUE
                        queue [batch]
  -w WALLTIME, --walltime WALLTIME
                        walltime [24:00:00]
  -t TEMPLATE, --template TEMPLATE
                        script template
  -o OUTFILE, --outfile OUTFILE
                        output script
  -v, --verbose         verbosely print information. -vv for just printing
                        command not creating scripts and submitting jobs

Note: if "{}" appears in a command, it will be replaced with the current
filename. More format supported: "{%}" for basename, "{^suffix}" for clipping
"suffix", "{%^suffix}" for clipping suffix from basename. See more:
https://github.com/shenwei356/easy_qsub
```

splitdir

```
usage: cluster_files [-h] [-o OUTDIR] [-p PATTERN] [-k] [-m] [-f] indir

clustering files by regular expression [V3.0]

positional arguments:
  indir                 source directory

optional arguments:
  -h, --help            show this help message and exit
  -o OUTDIR, --outdir OUTDIR
                        out directory [<indir>.cluster]
  -p PATTERN, --pattern PATTERN
                        pattern (regular expression) of files in indir. if not
                        given, it will be the longest common substring of the
                        files. GROUP (parenthese) should be in the regular
                        expression. Captured group will be the cluster name.
                        e.g. "(.+?)_\d\.fq\.gz"
  -k, --keep            keep original dir structure
  -m, --mv              moving files instead of creating symbolic links
  -f, --force           force file overwriting, i.e. deleting existed out
                        directory

```


## Examples
    
1) Submit a single job

    easy_qsub 'ls -lh'

2) Submit multiple jobs, runing fastqc for a lot of fq.gz files

    easy_qsub -n 8 -m 2GB 'mkdir -p QC/{%^.fq.gz}.fastqc; zcat {} | fastqc -o QC/{%^.fq.gz}.fastqc stdin' *.fq.gz
   
Excuted commands are:

	mkdir -p QC/read_1.fastqc; zcat read_1.fq.gz | fastqc -o QC/read_1.fastqc stdin
	mkdir -p QC/read_2.fastqc; zcat read_2.fq.gz | fastqc -o QC/read_2.fastqc stdin

3) Supposing a directory ```rawdata``` containing **paired files** as below. 
    
    $ tree rawdata
    rawdata
    ├── A2_1.fq.gz
    ├── A2_1.unpared.fq.gz
    ├── A2_2.fq.gz
    ├── A2_2.unpared.fq.gz
    ├── A3_1.fq.gz
    ├── A3_1.unpared.fq.gz
    ├── A3_2.fq.gz
    ├── A3_2.unpared.fq.gz
    └── README.md


And I have a program ```script.py```, which takes a directory as input and do some thing
with the **paired files**. Command is like this, ```script.py dirA```.

It is slow by submiting jobs like example 2), handing A2_\*.fq.gz 
and then A3_\*.fq.gz. So we can split ```rawdata``` directory into multiple directories, and
submit jobs for all directories.

	cluster_files -p '(.+?)_\d\.fq\.gz' rawdata
	
    tree rawdata.cluster/
    rawdata.cluster/
    ├── A2
    │   ├── A2_1.fq.gz -> ../../rawdata/A2_1.fq.gz
    │   └── A2_2.fq.gz -> ../../rawdata/A2_2.fq.gz
    └── A3
        ├── A3_1.fq.gz -> ../../rawdata/A3_1.fq.gz
        └── A3_2.fq.gz -> ../../rawdata/A3_2.fq.gz

	easy_qsub 'script.py {}' rawdata.split/*
	
4) Another example

    tree rawdata2
    rawdata2
    ├── OtherDir
    │   └── abc.fq.gz.txt
    ├── S1
    │   ├── A2_1.fq.gz
    │   ├── A2_1.unpared.fq.gz
    │   ├── A2_2.fq.gz
    │   ├── A2_2.unpared.fq.gz
    │   ├── A4_1.fq.gz
    │   └── A4_2.fq.gz
    └── S2
        ├── A3_1.fq.gz
        ├── A3_1.unpared.fq.gz
        ├── A3_2.fq.gz
        └── A3_2.unpared.fq.gz
    
    cluster_files -p '(.+?)_\d\.fq\.gz' rawdata2/
    
    tree rawdata2.cluster/
    rawdata2.cluster/
    ├── A2
    │   ├── A2_1.fq.gz -> ../../rawdata2/S1/A2_1.fq.gz
    │   └── A2_2.fq.gz -> ../../rawdata2/S1/A2_2.fq.gz
    ├── A3
    │   ├── A3_1.fq.gz -> ../../rawdata2/S2/A3_1.fq.gz
    │   └── A3_2.fq.gz -> ../../rawdata2/S2/A3_2.fq.gz
    └── A4
        ├── A4_1.fq.gz -> ../../rawdata2/S1/A4_1.fq.gz
        └── A4_2.fq.gz -> ../../rawdata2/S1/A4_2.fq.gz
    
    cluster_files -p '(.+?)_\d\.fq\.gz'  rawdata2/ -k -f  # keep original dir structure 
    
    tree rawdata2.cluster/
    rawdata2.cluster/
    ├── S1
    │   ├── A2
    │   │   ├── A2_1.fq.gz -> ../../../rawdata2/S1/A2_1.fq.gz
    │   │   └── A2_2.fq.gz -> ../../../rawdata2/S1/A2_2.fq.gz
    │   └── A4
    │       ├── A4_1.fq.gz -> ../../../rawdata2/S1/A4_1.fq.gz
    │       └── A4_2.fq.gz -> ../../../rawdata2/S1/A4_2.fq.gz
    └── S2
        └── A3
            ├── A3_1.fq.gz -> ../../../rawdata2/S2/A3_1.fq.gz
            └── A3_2.fq.gz -> ../../../rawdata2/S2/A3_2.fq.gz

## Copyright

Copyright (c) 2015, Wei Shen (shenwei356@gmail.com)

[MIT License](https://github.com/shenwei356/easy_qsub/blob/master/LICENSE)