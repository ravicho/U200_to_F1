# Porting Applications from Alveo U200 to AWS F1 with Vitis

This article explains how to port a Vitis application developed for an Alveo U200 card to an AWS EC2 F1 instance. 

The Vitis development flow provides platform independent APIs and interfaces to the developer. This greatly facilitates the process of migrating applications across similar FPGA acceleration cards. In this example, the source code for the software program and for the FPGA kernels remains unchanged. Only command line changes are necessary to port the application from Alveo U200 to AWS F1.

You will start with an existing project built for U200 and review the compilation script. Next, you will modify the script to target the same application on AWS F1. Finally, you will generate the Amazon FPGA Image (AFI) that can be loaded on the AWS F1 instance. 


## Overview of the Application 

A Vitis FPGA-accelerated application consists of a software program running on a host CPU which interacts with one or more accelerators running in a Xilinx FPGA device. 

In this lab, the accelerator is a simple vector-add kernel. 

The `src` directory contains the source code for the project:

- `vadd.cpp` contains the C++ source code of the accelerator which adds 2 arbitrarily sized input vectors. 
- `host.cpp` contains the main function running on the host CPU. The host application is written in C++ and uses OpenCL™ APIs to interact with the FPGA accelerator.

The  `u200` and `f1` directories contain the Makefiles and scripts for building for Alveo U200 and AWS F1 respectively.

## Building for Alveo U200

1. Go to the `u200` directory 

2. The host program is built with the following command:

    ```
    g++ -D__USE_XOPEN2K8 -I/$XILINX_XRT/include/ -I./src -O3 -Wall -fmessage-length=0 -std=c++11 ../src/host.cpp -L/$XILINX_XRT/lib -lxilinxopencl -lpthread -lrt -o build_u200/host
    ```

    This command ensures that the program is linked with the Xilinx Runtime (XRT) libraries. XRT provides platform-independent APIs to interact with the FPGA, allowing the source code to remain the same for U200 and F1.

3. The FPGA binary is built with the following commands:

    ```
    v++ -c -g -t hw -R 1 -k vadd --config ./options.cfg --profile_kernel data:all:all:all --profile_kernel stall:all:all:all --save-temps --temp_dir ./temp_dir --report_dir ./report_dir --log_dir ./log_dir -I../src ../src/vadd.cpp -o ./vadd.hw.xo
    v++ -l -g -t hw -R 1 --config ./options.cfg --profile_kernel data:all:all:all --profile_kernel stall:all:all:all --temp_dir ./temp_dir --report_dir ./report_dir --log_dir ./log_dir -I../src vadd.hw.xo -o add.hw.xclbin
    ```

    The Vitis v++ compiler is used to build the FPGA binary. The first command (`v++ -c`) compiles the source code for vector-add kernel (vadd.cpp) and generates the compiled kernel object (.xo). The second command (`v++ -l`) links the compiled kernel to the shell and produces the FPGA binary (.xclbin file) which can then be loaded on the U200 acceleration card.
    
    For both commands, the `--config` option is used to pass the name of a file called `options.cfg` containing additional options specific to building for U200.
    
4. The `options.cfg` file contains the following options: 

    ```
    platform=xilinx_u200_xdma_201830_2
    [connectivity]
    sp=vadd_1.in1:DDR[1]
    sp=vadd_1.in2:DDR[1]
    sp=vadd_1.out:DDR[1]
    ```

    The `platform` option specifies which acceleration platform is targeted for the build. Here we are using the U200 shell.
    
    The `sp` options are used to specify the assignment of kernel arguments to DDR banks. In this case, we are mapping all three kernel arguments to DDR[1], which is the DDR interface located in the shell on Alveo U200.
    
    Putting all the platform-specific options in one file is not mandatory but it is very convenient and facilitates the porting process. With this approach, the main command line can be reused as is for all platforms.
     
    Refer to the [Vitis Documentation](https://www.xilinx.com/html_docs/xilinx2020_1/vitis_doc/kme1569523964461.html) for more information on v++ related commands and options. 

5. The Makefile provided in the directory can be used to build the project for U200. Running `make build` will build the host application, compile the kernel and finally create the `vadd.xclbin` for U200. 
    
    
## Building for AWS F1

In order to port the vector-add example from Alveo U200 to AWS F1, the only change required is in the `options.cfg` file. The source code remains unchanged and the g++ and v++ commands remain identical.

1. Go to the `f1` directory 

2.  The `options.cfg` file for AWS F1 contains the following options:

    ```
    platform=xilinx_aws-vu9p-f1_shell-v04261818_201920_2
    [connectivity]
    sp=vadd_1.in1:DDR[0]
    sp=vadd_1.in2:DDR[0]
    sp=vadd_1.out:DDR[0]
    ```
    
    The `platform` option is set to target the AWS F1 shell. The string used corresponds to the name of the latest shell which can be found [here](https://github.com/aws/aws-fpga/tree/master/Vitis/aws_platform) on the aws-fpga repo.

    The `sp` options are set to connect the kernel arguments to DDR[0], which is the DDR interface located in the AWS F1 shell. Keeping the same settings as the U200 would produce a working design on AWS F1. But in order to produce exactly the same configuration and target the DDR interface located in the AWS F1 shell, the sp options are modified to use DDR[0].
    
    These changes are the only ones needed to port this project from U200 to F1. 
    
3.  You can build the project for AWS F1 using the exact same commands that were used for U200:

    ```
    export PLATFORM_REPO_PATHS=/home/centos/src/project_data/aws-fpga/Vitis/aws_platform
    
    g++ -D__USE_XOPEN2K8 -I/$(XRT_PATH)/opt/xilinx/xrt/include/ -I./src -O3 -Wall -fmessage-length=0 -std=c++11 ../src/host.cpp -L/$(XRT_PATH)/opt/xilinx/xrt/lib/ -lxilinxopencl -lpthread -lrt -o build_u200/host```

    v++ -c -g -t hw -R 1 -k vadd --config ./options.cfg --profile_kernel data:all:all:all --profile_kernel stall:all:all:all --save-temps --temp_dir ./temp_dir --report_dir ./report_dir --log_dir ./log_dir -I../src ../src/vadd.cpp -o ./vadd.hw.xo
    v++ -l -g -t hw -R 1 --config ./options.cfg --profile_kernel data:all:all:all --profile_kernel stall:all:all:all --temp_dir ./temp_dir --report_dir ./report_dir --log_dir ./log_dir -I../src vadd.hw.xo -o add.hw.xclbin    
    ```
    
    The PLATFORM_REPO_PATHS environment variable is used to specify the directory where the AWS platform (xilinx_aws-vu9p-f1_shell-v04261818_201920_2) is installed.
    
    NOTE: Alternatively, you can use the provided Makefile to build the project:
    
     ```
    export PLATFORM_REPO_PATHS=/home/centos/src/project_data/aws-fpga/Vitis/aws_platform
    make build
    ```   
    
4. When targeting AWS F1, you need to go through the additional step of creating an Amazon FPGA Image (AFI). This is done with the `create_vitis_afi.sh` command provided by AWS. More information about this command is available on the [AWS documentation](https://github.com/aws/aws-fpga/blob/master/Vitis/README.md#2-create-an-amazon-fpga-image-afi).     
    
    Use the commands below to generate the `awsxclbin` file

    ``` 
    cd /home/centos/src/project_data; 
    git clone https://github.com/aws/aws-fpga.git                                         
    cd aws-fpga; source vitis_setup.sh
    cd /home/centos/src/project_data/U200_to_F1/f1
    $AWS_FPGA_REPO_DIR/Vitis/tools/create_vitis_afi.sh -xclbin=./vadd.xclbin -o=./vadd -s3_bucket=<bucket-name> -s3_dcp_key=f1-dcp-folder -s3_logs_key=f1-logs 
    ```

    NOTE: Make sure to use your S3 bucket information when running the `create_vitis_afi` command.

    This command creates an *_afi_id.txt file. Open this file and record the AFI ID. Check the AFI creation status using AFI ID as shown below

    ```aws ec2 describe-fpga-images --fpga-image-ids <AFI ID>```
    
    When the state is shown as 'available', it indicates AFI creation is completed.  

    ``` 
        "State":  {
            "Code": "available" 
        },
    ```

## Running the Application on AWS F1

1. Execute the following command to source the Vitis runtime environment 

    ```source $AWS_FPGA_REPO_DIR/vitis_runtime_setup.sh```

2. Execute the host application with the .awsxclbin FPGA binary

    ``` ./vadd vadd.awsxclbin ``` 
    
3. The messages below will indicate that the program ran successfully.

    ```
    Found Platform
    Platform Name: Xilinx
    INFO: Reading ./vadd.awsxclbin
    Loading: './vadd.awsxclbin'
    TEST PASSED
    ```

## Conclusion 

In the Vitis flow, the user can develop source code which remains mostly agnostic of platform-specific platform details. This greatly facilitates the process of migrating applications across similar FPGA acceleration cards. In this example, the same source code could be ported from Alveo U200 to AWS F1 without any changes at all. Only a couple of changes to the v++ compilation options were required.
