UNVMe - A User Space NVMe Driver Project
========================================

UNVMe is a user space NVMe driver developed at Micron Technology for the
purpose of benchmarking different polling models in comparison to the
kernel space NVMe driver interupt driven model.

The UNVMe driver depends on features provided by the vfio module in the
Linux kernel (introduced since 3.6).

From the application perspective, UNVMe provides a single library (libunvme.a)
with a set of custom APIs for accessing NVMe devices.

In this project, UNVMe is implemented as three distinct models:

    model_apc   -   Using this library model, the application will have
                    exclusive access to an NVMe device, where the application
                    will process the completion queue at the time of checking
                    for I/O status.

    model_tpc   -   Using this library model, the application will have
                    exclusive access to an NVMe device, where a built-in
                    thread pool will automatically process IO completion,
                    and the application can later check for I/O status.

    model_cs    -   Using the client/server model, the driver will run
                    as a background service to manage a set of NVMe devices
                    and receive requests from client applications.
                    An application which runs in a separate process can
                    send requests to the driver.  This model is similar to
                    the traditional driver model which allows one or more
                    applications to access a device.

A selected model can be built by specifying the named target.
All three models export the same set of library APIs.

It should be noted that the UNVMe interface is designed to allow the user
to create specified number of I/O submission-completion queue pairs.
The queues will be created exclusively for and when a session is opened.
For multi-threaded applications, an application thread can perform I/O on
one or more queues, but a queue must only be accessed by a single thread
in order to guarantee thread-safe operations.


System Requirements
===================

UNVMe has only been built and tested on CentOS 6 and 7 running on x86_64 CPU
based systems with vfio support which requires the following features:

    VT-d        -   Hardware must have Intel CPU that supports VT-d
                    (Virtualization Technology for Directed I/O).
                
    vfio        -   Linux kernel 3.6 or later, and the kernel must be built
                    with these configurations enabled:

                        CONFIG_IOMMU_API=y
                        CONFIG_IOMMU_SUPPORT=y
                        CONFIG_INTEL_IOMMU=y
                        CONFIG_VFIO=m
                        CONFIG_VFIO_PCI=m
                        CONFIG_VFIO_IOMMU_TYPE1=m

                    Also the boot command line must have the "intel_iommu=on"
                    argument set (and optionally "iommu=pt").


Build and Run
=============

A specific model can also be built by explicitly specifying the one of the
following targets:

    $ make model_apc
    $ make model_tpc
    $ make model_cs

The Makefile.def file is provided for the user to specify the default
model to build when running:

    $ make

Make will keep track of the last model built and will default to that model
before using the DEFAULT_MODEL setting in Makefile.def.

When building model_apc or model_tpc, make will produce a 'libunvme.a'
library.  When building the model_cs, make will produce a 'libunvme.a' 
library and a 'unvme' executable which is the driver to be run as a service.


If the model_cs is built, then the service needs to be run specifying
a list of mapped vfio devices as its arguments, i.e. prior to applications.
For example:

    $ src/unvme /dev/vfio/X /dev/vfio/Y ...

The UNVMe runtime log messages will be saved in /dev/shm/unvme.log.


Before running the UNVMe service or applications, the user must first run
the setup script once to bind the NVMe to vfio devices using the following
command:

    $ test/unvme-setup

To reset the devices to the default NVMe kernel space drivers, run command:

    $ test/unvme-setup reset


Applications and Tests
======================

Applications can be built using the libunvme.h and libunvme.a interfaces.
The library provides APIs for both UNVMe and direct NVMe command access
with vfio functions support for DMA memory allocation.  

Examples of both UNVMe and NVMe level testing are provided under the test
directory.

UNVMe test applications can be run as:

    $ test/unvme/unvme_api_test /dev/vfio/X
    $ test/unvme/unvme_mts_test /dev/vfio/X
    ...


NVMe level tests (bypassing UNVMe driver) can be run as:

    $ test/nvme/nvme_identify /dev/vfio/Z          # identify controller
    $ test/nvme/nvme_identify /dev/vfio/X 1        # identify namespace 1
    $ test/nvme/nvme_get_log_page /dev/vfio/X 1 1  # get log page error info
    $ test/nvme/nvme_get_log_page /dev/vfio/X 1 2  # get log page SMART health
    $ test/nvme/nvme_get_log_page /dev/vfio/X 1 3  # get log page firmware slot
    ...


Benchmark
=========

UNVMe has been benchmarked with fio (Flexible I/O Tester) using the test
script test/unvme-benchmark.  The ioengine/unvme_fio is provided for
this purpose.  In order to build the unvme_fio engine, the user must either
set the FIO_DIR variable to point to the fio source path in the Makefile.def
or specify FIO_DIR on the make command line as:

    $ FIO_DIR=/path/fio make model_apc


To produce benchmark results for UNVMe model_apc, run:

    $ test/unvme-setup  # only need to be done once if has not
    $ make model_apc
    $ OUTDIR=out/apc test/unvme-benchmark /dev/vfio/X

To produce benchmark results for UNVMe model_tpc, run:

    $ make model_tpc
    $ OUTDIR=out/tpc test/unvme-benchmark /dev/vfio/X

To produce benchmark results for UNVMe model_cs, run:

    $ make model_cs
    $ src/unvme /dev/vfio/X ...
    $ OUTDIR=out/cs test/unvme-benchmark /dev/vfio/X


To reset and get benchmark results for the NVMe kernel space driver, run:

    $ test/unvme-setup reset
    $ OUTDIR=out/nvme test/unvme-benchmark /dev/nvme0n1


Notes:

    + The unvme-benchmark script will run fio tests for random read and
      random write using 1, 4, 8, and 16 jobs (threads) and iodepth of
      1, 4, 8, 16, 32, and 64.

    + For UNVMe, the number of fio jobs will be translated to number of
      queues and the queue size is set to iodepth + 1.

    + The default output directory will be "out" relative to the test
      directory and can be overriden by specifying OUTDIR on the
      shell commandl ine.

    + If FIO_DIR is not set in Makefile.def then the FIODIR variable
      must be specified on the shell command line.  

    + If nsid is other than 1, the variable NSID must be specified on the
      shell command line.


Documentation
=============

Doxygen formatted documents for the source code can be generated as:

    $ make doc

The output HTML based document can then be browsed from doc/html/index.html.
