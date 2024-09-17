# FSE25CFPOFuzz
## Artifact description
This is the artifact of submission #804 (**Context-Free Property Oriented Fuzzing**) in FSE 2025. 

The artifact is provided as a docker image, which contains the prototype of the proposed method and benchmarks used for evaluation. The aim is to assist the reviewers in reproducing the experimental results in our evaluation.

## Prerequisites
+ Hardware: ~3.1 GHz CPU (all experiments were performed on a server with Intel(R) Xeon(R) Gold 6458Q CPU @ 3.10GHz, 128 Cores, 256GB), 500+GB free disk space.
+ Unix / Linux OS: We have validated the artifact on Ubuntu system
+ [Docker](https://www.docker.com/pricing/)

## Running the artifact
We assume that the following commands are run in sudo mode. 

Firstly, pull the already prebuilt docker image from [docker hub](https://hub.docker.com/r/fse25/cfpofuzz). Please make sure its name is `fse25/cfpofuzz`.
```sh
$ docker pull docker push fse25/cfpofuzz:v1
```

If everything is ok, a `fse25/cfpofuzz` image should be found in the images listed by `docker images` command. Then, you can create a container of such image and start a Bash session using the following command. An interactive `bash` shell on the container is also executed at the moment.
```sh
$ docker run -it fse25/cfpofuzz:v1 bash
```

If all goes well, the container should be running successfully. Otherwise, you can seek help from [Docker Doc](https://docs.docker.com/) if needed. 

Now, navigate into the directory containing our experimental environment and list the contents. 
```sh
$ cd /home/ubuntu/ && ls
CFPOfuzz /  CFPOfuzz-Benchmark / Fuzzdata /  Newbug / LLVM / demo /

$ cd /home/ubuntu/CFPOfuzz && ls
AFL-cfpo /  antlr  /  ccmop  /  rv-monitor /  wcompiler / AFLnet-cfpo / aspectc / examples / wac-pass

$ cd /home/ubuntu/CFPOfuzz-benchmark && ls
Exiv2 / Live555 / Mujs / OpenSSL / TinyDTLS / luna / run.py

$ cd /home/ubuntu/Fuzzdata && ls
data / data1 / gen_table.py / table.tex

$ cd /home/ubuntu/Newbug && ls
Live555 / Luna / Mujs / NewCrash / TinyDTLS

```
The necessary items to reproduce our evaluation are listed, including benchmark programs and scripts. The rest of this README assumes you are working in `/home/ubuntu`.
## Run the example for Motivition
In the example discussed in our paper, we utilized a simple C++ program to demonstrate the superiority of our approach. In most cases, our method is able to identify the target input that triggers the bug Around ***30 seconds***, whereas the coverage-guided (COV) approach struggles to find the target input within ***10 minutes***. ***Please note that we also use the monitor-instrumented version of the program for coverage-oriented fuzzing（COV）***。Next, we will use this simple demo to familiarize ourselves with our methodology and tools.
```
$ cd /home/ubuntu/demo
$ ls
in   / stack.mop   /  test.cpp
```
其中stack.mop的内容如下：
```
Stack() {
	event push after() :
              call(%push(...)){}
    event pop after() :
              call(% pop(...)){}
   cfg :
        S -> push S pop | S S |push S| epsilon
    @fail
    {
        printf("Stack hava errs !");
    }
}
```
In this context, ```event``` refers to events related to program behavior. Here, two events are defined: ```push ```and``` pop```. The ```cfg``` describes a property based on a context-free grammar, where ```fail``` indicates that a bug will be triggered if the program behavior violates the property, and ```match``` indicates that a bug will be triggered if the program behavior satisfies the property. Since our Motivition example only uses a single stack structure, there is no need to distinguish between different stacks. However, if multiple distinct stacks appear in the program, we can use a parameterized MOP file, where different stack base addresses are used as parameters for identification and differentiation. An example is as follows:
```
Stackp(std::stack<int> key) {
     event push after(std::stack<int> key): call(% stack<int>.push(...))&& target(key){}
     event pop after(std::stack<int> key): call(% stack<int>.pop(...))&& target(key){}
     cfg :
        S -> push S pop | S S | push S | epsilon
    @fail
    {
        printf("Stack hava errs at %p!",key);
        __RESET;
    }
}
```


Next, based on the MOP file, we perform the instrumentation and compilation of the program. Our instrumentation process is divided into three parts: ***event instrumentation***, ***related control condition instrumentation*** (associated with events), and AFL's original ***coverage instrumentation***. You can complete the entire instrumentation and compilation process by running the following command in the current directory (```/home/ubuntu/demo```):
```
$ wac -mop stack.mop -np -afl  -cxx test.cpp -o test <-print>
```

After the instrumentation and compilation are completed, a directory called ```stackaspect``` will be generated in the current directory. This directory contains relevant control flow dominator information, the generated runtime libraries, and other related data. Therefore, if you want to use our method for testing, you must include the path to this directory as a parameter. You can run the following command to perform the fuzzing test:
```
/home/ubuntu/CFPOfuzz/AFL-cfpo/afl-fuzz -c 1 -r ./stackaspect -i in -o out-cfpo ./test
```
The``` -c ```parameter is followed by different configuration modes, which is an integer between 1 and 7(***1: CFPO*+IP; 2: CFPO*+IP+COV; 3: CFPO*; 4: CFPO*+COV; 5: COV+IP; 6: CFPO+IP; 7: CFPO***)，The value 1 indicates the use of the target method proposed in this paper.

For property matching, distance calculation, and other related information, we use*** shared memory***  for communication. Determining ***whether the defined receiving state*** has been reached is done solely through the information in shared memory. Therefore, in coverage-guided fuzzing, the information in the stackaspect directory is not needed. The fuzzing command is as follows:
```
/home/ubuntu/CFPOfuzz/AFL-cfpo/afl-fuzz  -i in -o out-afl ./test
```

The initial input is stored in``` /home/ubuntu/demo/in/input```, and its content is ```"hhhhhh"```.You will observe that our method will identify the ```target input``` that triggers the bug in approximately ```30 seconds```, and it will be saved in ```/home/ubuntu/demo/out-cfpo/RVC```. In contrast, coverage-guided fuzzing is unlikely to discover the target input within``` 10 minutes```.
## Reproducing experimental results
In our experiments, there are a total of 20 tasks. Among them, 13 tasks (from the `CFPOfuzz-Benchmark`) were evaluated under eight configurations (CFPO*+IP, CFPO+IP, CFPO*, CFPO, COV, COV+IP, COV+CFPO*, COV+IP+CFPO*). Our target approach is CFPO*+IP. Since it requires running 104 tasks in parallel, we provide scripts to automate and execute all experiments. The remaining 7 tasks(from the `New-bug`) involve previously unknown zero-day bugs discovered in real-world programs, which you can reproduce individually using our method as described in this section.

Given the long runtime of the evaluation and the large number of parallel tasks, all of our experiments were conducted on a server with 128 CPU cores and 256GB of memory.

### Reproduce the experimental results of existing bugs:
```bash
$ cd /home/ubuntu/CFPOfuzz-benchmark
# python3 run.py  execute time   test rounds
$ python3 run.py  24  5
# wait......
```


All tasks will be repeated 5 times, requiring approximately **12480 core-hours**. If you run one CPU core for one hour, that is one core-hour. 

After the fuzzing is completed, The time to discover the first target input is redirected to the output at `/home/ubuntu/Fuzz-data/data.`The directory structure of Fuzz-data is roughly as follows.
```
├── data
│   ├── num1
│   │   ├── exiv2
│   │   │   └── exiv2-CVE-2023-44398
│   │   │       ├── out1
│   │   │       ├── out2
│   │   │       ├── out3
│   │   │       ├── out4
│   │   │       ├── out5
│   │   │       ├── out6
│   │   │       ├── out7
│   │   │       └── out8
│   │   ├── Live555
│   │   │   ├── Live555-CVE-2013-6933
│   │   │   └── Live555-CVE-2019-7314
|   |   |   .......
|   ├── num2
|   ├── num3
|   ├── num4
|   ├── num5
|——gen_table.py
```
In this context, num**x** represents the results of the **x-th** experiment. The subdirectories under num correspond to different programs, and within each program directory, specific task names (including CVE identifiers) are located. As mentioned earlier, we conducted experimental evaluations using eight different configurations for each task. Therefore, each task contains eight out directories, each with a distinct identifier. These identifiers represent different configurations, recording the time taken to discover the first target input for the same task under different configurations(out1:CFPO*+IP; out2：CFPO*+IP+COV; out3:CFPO*; out4:CFPO*+COV; out5:COV+IP; out6:CFPO+IP; out7:CFPO; out8:COV) 
After completing five rounds of experiments, you can collect, organize, and compute the experimental data using the following command. The final results will be presented in the **table.tex** file.

```
$cd /home/ubuntu/Fuzz-data
$python3 gen_table.py
```

### Reproduce the experimental results of new bugs:
In the experiment, we discovered seven previously unknown zero-day bugs (LN1, LN2, LN3, MJ5, LV3, LV4, TD4) in real-world programs. These bugs include types such as floating-point exception, null pointer dereference, out-of-bounds access, infinite loop, and denial of service. In```/home/ubuntu/New-bug```, we have stored the source code for all the zero-day bugs, the inputs that trigger the bugs, and our abstracted event properties along with the results after instrumentation. Specifically, the inputs that trigger the bugs are located in ```/home/ubuntu/New-bug/NewCrash```. Following the guidance in this section, you can use our method to discover the inputs that trigger these bugs one by one.

Using ```LN3``` as an example, we first navigate to the source code directory of ```LN3```.
```
$cd /home/ubuntu/Newbug/Luna/luna-cfg3
```

You will see our MOP file with the following content (``luna.mop``).
```
luna(){
	event A after() :
              call(% RK_Cnot0(...)){}
    event B after() :
              call(% LUNA_OP_MOD(...)){}
    event C after() :
              call(% LUNA_OP_DIV(...)){}
	    cfg: S → Q P | S P | S S,
             P → B | C,
             Q → A Q P | P Q A | Q Q | epsilon
        @match {
		std::cout<<"error occurs!"<<std::endl;
        }
}
```
In this context, we defined three events related to program behavior, labeled as A, B, and C. The property is expressed as follows: if the number of occurrences of event A is less than the sum of occurrences of events B and C, a bug will be triggered. We also instrumented and compiled the program based on the description in MOP. It is important to note that, unlike the instrumentation and compilation command in the ```Motivition demo``` above, here we are instrumenting the entire Luna project. Therefore, the following command is required for instrumentation and compilation of the entire project.
```
$ wac -proj test_build.json -afl -np <-print>
```
The parameter ```-print``` is optional and can be used to output the instrumented source code (as .i files). The file ```test_build.json``` is the configuration file for instrumenting and compiling the entire project, and its content is as follows:
```
{
    "project_name": "luna",
    "compile_language": "c",
    "project_root_dir": "./",
    "mop_file": "luna.mop",
    "project_build_subdir": "",
    "project_compile_cmd": "make"
}
```
After the instrumentation and compilation are completed, a ```lunaaspect``` folder is generated in the current directory. This folder contains the runtime libraries we created, control flow dominator information, event lists, and other relevant data. Additionally, the instrumented ``luna ``executable is also generated. We will proceed with fuzzing using the following command.
```
$ /home/ubuntu/CFPOfuzz/AFL-cfpo/afl-fuzz -c 1 -r ./lunaaspect -i in -o out1 ./luna @@
```
Here, compared to standard fuzzing, two additional parameters, ```-c ```and ```-r```, are introduced, which have similar meanings as in the demo test mentioned earlier. You can also use the``` -k ```parameter to set the hyperparameter K (otherwise, the default value of 8 is used), and the ```-u``` parameter to output the time it took to discover the first target input to a specified directory. After fuzzing completes, if a target input is found, it will be stored in the``` out1/RVC ```directory.

It is important to note that if you are testing protocol-based software, you must use AFLnet-cfpo located at ```/home/ubuntu/CFPOfuzz/AFLnet-cfpo/afl-fuzz```. The main difference from the parameters mentioned earlier is that ```-c``` is replaced with ```-v```. You can select the fuzzing mode using``` -v x```, where``` x``` is an integer between 1 and 7 (***1: CFPO*+IP; 2: CFPO*+IP+COV; 3: CFPO*; 4: CFPO*+COV; 5: COV+IP; 6: CFPO+IP; 7: CFPO***). The parameters are similar to those used in standard AFLnet. If you want to run fuzzing guided purely by coverage (COV), the command is the same as for AFL or AFLnet. It is worth noting that even for COV-guided fuzzing(AFL or AFLnet), we have modified them to incorporate our property matching. As long as the program behavior reaches the defined receiving state, it will be treated as a crash and stored accordingly.

The reproduction process for the other New-bug cases is similar to the one described above. We have provided the fuzzing commands for all tasks in the GitHub repository. You can use our method to reproduce them one by one. Additionally, for the previously mentioned bugs with existing CVE identifiers, you can follow the same process and refer to the fuzzing commands we provided for individual testing.
