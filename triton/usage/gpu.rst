=============
GPU Computing
=============

Overview
========

Triton has GPU cards from two different Nvidia generations. SL390s G7
nodes with Fermi generation cards (M2050,M2070,M2090) and PowerEdge
C4130 servers with newer Kelper and Pascal generations of K80/P100 cards.

Hardware breakdown
==================

+---------------+----------------+--------------+----------------+---------------------------+-------------------+---------------------------+----------------------+
| Card          | total amount   | nodes        | architecture   | compute threads per GPU   | memory per card   | CUDA compute capability   | Slurm feature name   |
+===============+================+==============+================+===========================+===================+===========================+======================+
| Tesla M2090   | 22             | gpu[1-11]    | Fermi          | 512                       | 6G                | 2.0                       | m2090                |
+---------------+----------------+--------------+----------------+---------------------------+-------------------+---------------------------+----------------------+
| Tesla M2070   | 6              | gpu[17-19]   | Fermi          | 448                       | 6G                | 2.0                       | m2070                |
+---------------+----------------+--------------+----------------+---------------------------+-------------------+---------------------------+----------------------+
| Tesla M2050   | 10             | gpu[12-16]   | Fermi          | 448                       | 3G                | 2.0                       | m2050                |
+---------------+----------------+--------------+----------------+---------------------------+-------------------+---------------------------+----------------------+
| Tesla K80\*   | 12             | gpu[20-22]   | Kepler         | 2x2496                    | 2x12GB            | 3.7                       | teslak80             |
+---------------+----------------+--------------+----------------+---------------------------+-------------------+---------------------------+----------------------+
| Tesla P100    | 20             | gpu[23-27]   | Pascal         | 3854                      | 16GB              | 6.0                       | teslap100            |
+---------------+----------------+--------------+----------------+---------------------------+-------------------+---------------------------+----------------------+

* Note: Tesla K80 cards are in essence two GK210 GPUs on a single chip

Detail info about cards available at
http://en.wikipedia.org/wiki/Nvidia_Tesla and for general info about
Nvidia GPUs
http://www.nvidia.com/object/tesla-supercomputing-solutions.html

Using GPU nodes
===============

GPU partitions
--------------

There are two queues governing these nodes; gpu and gpushort, where the
latter is for jobs up to 4 hours.

The latest details can always be found with the following command:

::

    $ slurm p | grep gpu

GPU node allocation
-------------------

For gpu resource allocation one has to request a gpu partition, with 
``-p gpu`` ,  and a GPU resource, with  ``--gres=gpu:N`` , where \ ``N``
stands for number of requested GPU cards. To request a specific card,
one must use syntax  ``--gres=gpu:CARD_TYPE:N`` ,  see 'Slurm feature
name' in the table above.

.. raw:: html

   <div class="line number1 index0 alt2">

 

::

    -p gpu --gres=gpu:2  # any two gpu cards on gpu partition
    -p gpushort --gres=gpu:teslak80:1   # one tesla card on gpushort

For the full current list of configured SLURM gpu cards names run 
``slurm features``.

.. raw:: html

   </div>

GPU nodes environment and CUDA
------------------------------

User environment on ``gpu*`` nodes is the same as on other nodes, the
only difference is that they have nvidia kernel modules for Tesla cards.
`CUDA <http://www.nvidia.com/object/cuda_home_new.html>`__ comes through
``module``.

::

    $ module avail CUDA    # list installed CUDA modules
    $ module load CUDA/7.5.18   # load CUDA environment that you need
    $ nvcc --version   # see actual CUDA version that you got

Running a GPU job in serial
---------------------------

Quick interactive run

::

    $ module load CUDA
    $ srun -t 00:30:00 -p gpushort --gres=gpu:1 $WRKDIR/my_gpu_binary

Allocating a gpu node for longer interactive session, this will give you
a shell sessions

::

    $ module load CUDA
    $ sinteractive -t 4:00:00 -p gpushort --gres=gpu:1
    gpuXX$ .... run something
    gpuXX$ exit 

Run a batch job

::

    $ sbatch gpu_job.sh

Where ``gpu_job.sh`` is

::

    #!/bin/bash

    #SBATCH --time=01:15:00          ## wallclock time hh:mm:ss
    #SBATCH -p gpu                   ## partition: gpu
    #SBATCH --gres=gpu:teslak80:1    ## one K80 requested

    module load CUDA

    ## run my GPU accelerated executable, note the --gres
    srun --gres=gpu:1  $WRKDIR/my_gpu_binary

Development
===========

Compiling
---------

In case you either want to compile a CUDA code or a code with GPU
support, you must do it on one of the gpu nodes (because of nvidia libs
installed on those nodes only).

::

    $ sinteractive -t 1:00:00 -p gpushort --gres=gpu:1    # open a session on a gpu node
    $ module load CUDA                                    # set CUDA environment
    $ nvcc cuda_code.cu -o cuda_code                      # compile your CUDA code
    .. or compile normally any other code with 'make'

Debugging
---------

CUDA SDK provides an extension to the well-known gnu debugger gdb. Using
cuda-gdb it is possible to debug the device code natively on the GPU. In
order to use the cuda-gdb, one has to compile the program with option
pair -g -G, like follows:

::

    $ nvcc -g -G cuda_code.cu -o cuda_code

See `CUDA-GDB User
Guide <http://developer.download.nvidia.com/compute/DevZone/docs/html/C/doc/cuda-gdb.pdf>`__
for a more information on cuda-gdb.

Applications and known issues
=============================

nvidia-smi utility
------------------

Could be useful for debugging, in case one want to see the actual gpu
cards available on the node. If this command returns an error, it is
time to report that something is wrong on the node.

::

    gpuxx$ nvidia-smi -L   # gives a list of GPU cards on the node

cuDNN
-----

``cudnn`` is available as a module. The latest version can be found with
``module spider cudnn``. Note that (at least the later versions of)
cudnn require newer cards and cannot be used on the old fermi cards.
E.g. tensorflow does not run on the older fermi cards for this reason.

Tensorflow example
------------------

This chapter gives a step-by-step guide how to run the tensorflow
cifar10 example on 4 gpu's. All commands below are typed on the login
node, it is not necessary to ssh to a gpu node first.

First load anaconda (python), CUDA and cudnn

::

    $ module load anaconda2 CUDA/7.5.18 cudnn/4

After that create a conda environment to install tensorflow in:

::

    $ conda create -n tensorflow python=2.7

    $ source activate tensorflow
    $ pip install --ignore-installed --upgrade 
    https://storage.googleapis.com/tensorflow/linux/gpu/tensorflow-0.8.0-cp27-none-
    linux_x86_64.whl
    $ pip install --upgrade 
    https://storage.googleapis.com/tensorflow/linux/gpu/tensorflow-0.8.0-cp27-none-
    linux_x86_64.whl

For some (unclear) reason you have to run the pip command twice, first
with ``--ignore-installed`` and second time without to make the conda
environment work.

Now we can create a batch script (``submit_cifar.sh``) that runs this
code on 4 gpus

::

    #!/bin/bash
     
    #Request 4 gpus
    #SBATCH --gres=gpu:teslak80:4
    #SBATCH -p gpushort
    #SBATCH --mem-per-cpu 10G
    #SBATCH -t 4:00:00

    module load anaconda2 CUDA/7.5.18 cudnn/4
    source activate tensorflow

    python -m tensorflow.models.image.cifar10.cifar10_multi_gpu_train --num-gpus 4

You can submit this job with

::

    $ sbatch submit_cifar.sh

and you'll be able to find the results in the slurm log file.

Theano configuration
--------------------

If you're using the theano library, you need to tell theano to store
compiled code on the local disk on the compute node. Create a file
~/.theanorc with the contents

::

    [global]
    base_compiledir=/tmp/%(user)s/theano

Also make sure that in your batch job script you create this directory
before you launch theano. E.g.

::

    mkdir -p /tmp/${USER}/theano

The problem is that by default the base\_compiledir is in your home
directory (~/.theano/), and then if you first happen to run a job on a
newer processor, a later job that happens to run on an older processor
will crash with an "Illegal instruction" error.

CUDA samples
------------

There are CUDA code samples provided by Nvidia that can be useful for a
sake of testing or getting familiar with CUDA. Placed
at \ ``$CUDA_HOME/samples``. To play with:

::

    $ sinteractive -t 1:00:00 -p gpushort --gres=gpu:1
    $ module load CUDA
    $ cp -r $CUDA_HOME/samples $WRKDIR
    $ cd $WRKDIR/samples
    $ make TARGET_ARCH=x86_64
    $ ./bin/x86_64/linux/release/deviceQuery
    ...
    $ ./bin/x86_64/linux/release/bandwidthTest
    ...

Attachments and useful links
============================

| `CUDA C Programming
  Guide <http://developer.download.nvidia.com/compute/DevZone/docs/html/C/doc/CUDA_C_Programming_Guide.pdf>`__
| `CUDA Zone on
  NVIDIA <http://developer.nvidia.com/category/zone/cuda-zone>`__
| `CUDA FAQ <http://developer.nvidia.com/cuda/cuda-faq>`__
