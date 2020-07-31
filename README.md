# Porting Application from U200 to F1

This article demonstrates how to port a Vitis application developed for U200 card to an F1 instance. 

You will start with an existing project that has been build for U200, review the compilation/linking script. Next, you will modify the scripts for targetting application on on F1. Finally, you will generate the AFI that can be run on the F1 instance. 

Run the following commands to clone the repository to your local workspace on F1 instance. 

``` 
mkdir /home/centos/src/project_data;
cd /home/centos/src/project_data; 
git clone https://github.com/ravicho/U200_to_F1
```


## Overview of the Application 

Vitis core development application consists of a software program running on a host CPU and interacting with one or more accelerators running on a Xilinx FPGA. In this lab, the accelerator has the functionality of a simple vector addition. The project comprises of multiple files under src directory.
- `vadd.cpp` contains the software implementation of the kernel which performs simple addition of 2 input values. 
- `host.cpp` contains the main function running on the host CPU. The host application is written in either C++ using OpenCL APIs calls to interact with the FPGA accelerator.
There are separate directories, `u200` and `f1` for building the kernel for `U200 card` and `F1 instance` respectively. 

## Build U200 Kernel

1. Change to directory `/home/centos/src/project_data/U200_to_F1/u200`
2. The software program is written in C/C++ and uses OpenCL™ API calls to communicate and control the accelerated kernels. Use the following command to compile the program using GCC/g++ compiler

    ```g++ -D__USE_XOPEN2K8 -D__USE_XOPEN2K8 -I/$(XRT_PATH)/opt/xilinx/xrt/include/ -I./src -O3 -Wall -fmessage-length=0 -std=c++11 ../src/host.cpp -L/$(XRT_PATH)/opt/xilinx/xrt/lib/ -lxilinxopencl -lpthread -lrt -o build_u200/host```

2. The following `v++ compile` command creates .xo file using standard v++ compile options to add profiling and location of directories for saving reports. 
    
    ```v++ -c -g -t hw -R 1 -k vadd --platform xilinx_u200_xdma_201830_2 --profile_kernel data:all:all:all --profile_kernel stall:all:all:all --save-temps --temp_dir ./temp_dir --report_dir ./report_dir --log_dir ./log_dir -I../src ../src/vadd.cpp -o ./vadd.hw.xo```
    
    - `-plaform` option sets the targetted plaform. For U200 card, its set to xilinx_u200_xdma_201830_2

3. The following `v++ linker` command creates xclbin which can be loaded on the FPGA. 

    ```v++ -l -g -t hw -R 1 --platform xilinx_u200_xdma_201830_2 --profile_kernel data:all:all:all --profile_kernel stall:all:all:all --temp_dir ./temp_dir --report_dir ./report_dir --log_dir ./log_dir  --config ./connectivity.cfg -I../src vadd.hw.xo -o add.hw.xclbin```

4. Kernel arguments are connected to the same bank DDR1 and `-sp` option is used to enable this linking. This option is added in connectivity section of `connectivity.cfg` file as shown below.
    ```[connectivity]
    sp=vadd_1.in1:DDR[1]
    sp=vadd_1.in2:DDR[1]
    sp=vadd_1.out:DDR[1]
    ```
    Refer to <a href="https://www.xilinx.com/html_docs/xilinx2020_1/vitis_doc/kme1569523964461.html"> Vitis Documentation </a>for more information on v++ related commands and options. These commands are encapsulated in Makefile targets. You can execute `make build` that will build the host, kernel and finally create the `vadd.xclbin` for u200. 
    
## Build F1 kernel - Compilation and Linking script changes 
Change to directory `/home/centos/src/project_data/U200_to_F1/f1` for building kernel for F1 instance. For this application to run on F1 instance, following changes are required. 

1. Modify the target platform to AWS platform by updatig v++ compile and linking scripts to the following 
    
    `-platform $(AWS_PLATFORM)`
    Refer<a href="https://github.com/aws/aws-fpga/tree/master/Vitis/aws_platform"> here</a> for the latest platform available.

2. Connect the kernel arguments connection to DDR[0] by updating `connectivity.cfg` as shown below
    ```[connectivity]
    sp=vadd_1.in1:DDR[0]
    sp=vadd_1.in2:DDR[0]
    sp=vadd_1.out:DDR[0]
    ```

    And that's all you need to modify in your application to port to F1. 
    
    You can execute the following command to build the host, kernel, and finally create the xclbin. 
    ```
    export PLATFORM_REPO_PATHS=/home/centos/src/project_data/aws-fpga/Vitis/aws_platform
    make build
    ```
    Next, you will need to create `awsxclbin` that can be loaded on F1 instance.

4. Run the following script to generate `awsxclbin`

    ``` 
    cd /home/centos/src/project_data; 
    git clone https://github.com/aws/aws-fpga.git                                         
    cd aws-fpga; source vitis_setup.sh
    cd /home/centos/src/project_data/U200_to_F1/f1
    $AWS_FPGA_REPO_DIR/Vitis/tools/create_vitis_afi.sh -xclbin=./vadd.xclbin -o=./vadd -s3_bucket=<bucket-name> -s3_dcp_key=f1-dcp-folder -s3_logs_key=f1-logs 
    ```

    -   The previous step will create *_afi_id.txt file. Open this file and record the `AFI Id`. Check the AFI creation status using AFI ID as shown below

        ```aws ec2 describe-fpga-images --fpga-image-ids <AFI ID>```
        -   If the state is shown as 'available', it indicates AFI creation is completed.  

            ``` 
            "State":  {
                "Code": "available" 
            },
            ```

## Run the Application on F1

1. Execute the following command to source the Vitis runtime environment 
```source $AWS_FPGA_REPO_DIR/vitis_runtime_setup.sh```
2. Execute the host application with the .awsxclbin FPGA binary
``` ./vadd vadd.awsxclbin ``` 
3. Above command displays that the Hardware run is Passing as shown below.
    ```
    Found Platform
    Platform Name: Xilinx
    INFO: Reading ./vadd.awsxclbin
    Loading: './vadd.awsxclbin'
    TEST PASSED
    ```

## Conclusion 

Its fairly straight forward to port your U200 application to F1 instance. In this article, you observed that there are only couple of scripting changes required to make the application compliant to F1 and absolutely no change required in source code at all. 

