在远程计算机上运行 Experiment
====================================

NNI 可以通过 SSH 在多个远程计算机上运行同一个 Experiment，称为 ``remote`` 模式。 这就像一个轻量级的训练平台。 在此模式下，可以从计算机启动 NNI，并将 Trial 并行调度到远程计算机。

远程计算机的操作系统支持 ``Linux``\ ， ``Windows 10`` 和 ``Windows Server 2019``。

必需组件
------------


* 
  确保远程计算机的默认环境符合 Trial 代码的需求。 如果默认环境不符合要求，可以将设置脚本添加到 NNI 配置的 ``command`` 字段。

* 
  确保远程计算机能被运行 ``nnictl`` 命令的计算机通过 SSH 访问。 同时支持 SSH 的密码和密钥验证方法。 高级用法请参考 `machineList part of configuration <../Tutorial/ExperimentConfig.rst>`__ 。

* 
  确保每台计算机上的 NNI 版本一致。

* 
  如果要同时使用远程 Linux 和 Windows，请确保 Trial 的命令与远程操作系统兼容。 例如，Python 3.x 的执行文件在 Linux 下是 ``python3``，在 Windows 下是 ``python``。

Linux
^^^^^


* 按照 `安装教程 <../Tutorial/InstallationLinux.rst>`__ 在远程计算机上安装 NNI 。

Windows
^^^^^^^


* 
  按照 `安装教程 <../Tutorial/InstallationLinux.rst>`__ 在远程计算机上安装 NNI 。

* 
  安装并启动 ``OpenSSH Server``。


  #. 
     在 Windows 上打开 ``设置`` 应用。

  #. 
     点击 ``应用``，然后点击 ``可选功能``。

  #. 
     点击 ``添加功能``，找到并选择 ``OpenSSH 服务器``，然后点击 ``安装``。

  #. 
     安装后，运行下列命令来启动服务并设为自动启动。

  .. code-block:: bat

     sc config sshd start=auto
     net start sshd

* 
  确保远程账户为管理员权限，以便可以停止运行中的 Trial。

* 
  确保除了默认消息外，没有别的欢迎消息，否则会导致 NodeJS 中的 ssh2 出错。 例如，如果在 Azure 中使用了 Data Science VM，需要删除 ``C:\dsvm\tools\setup\welcome.bat`` 中的 echo 命令。

  打开新命令窗口，如果输入如下，则表示正常。

  .. code-block:: text

     Microsoft Windows [Version 10.0.17763.1192]
     (c) 2018 Microsoft Corporation. All rights reserved.

     (py37_default) C:\Users\AzureUser>

运行实验
-----------------

例如， 有三台机器，可使用用户名和密码登录。

.. list-table::
   :header-rows: 1
   :widths: auto

   * - IP
     - Username
     - Password
   * - 10.1.1.1
     - bob
     - bob123
   * - 10.1.1.2
     - bob
     - bob123
   * - 10.1.1.3
     - bob
     - bob123


在这三台计算机或另一台能访问这些计算机的环境中安装并运行 NNI。

以 ``examples/trials/mnist-annotation`` 为例。 示例文件 ``examples/trials/mnist-annotation/config_remote.yml`` 的内容如下：

.. code-block:: yaml

   authorName: default
   experimentName: example_mnist
   trialConcurrency: 1
   maxExecDuration: 1h
   maxTrialNum: 10
   #choice: local, remote, pai
   trainingServicePlatform: remote
   # search space file
   searchSpacePath: search_space.json
   #choice: true, false
   useAnnotation: true
   tuner:
     #choice: TPE, Random, Anneal, Evolution, BatchTuner
     #SMAC (SMAC should be installed through nnictl)
     builtinTunerName: TPE
     classArgs:
       #choice: maximize, minimize
       optimize_mode: maximize
   trial:
     command: python3 mnist.py
     codeDir: .
     gpuNum: 0
   # machineList can be empty if the platform is local
   machineList:
     - ip: 10.1.1.1
       username: bob
       passwd: bob123
       # port can be skip if using default ssh port 22
       # port: 22
     - ip: 10.1.1.2
       username: bob
       passwd: bob123
     - ip: 10.1.1.3
       username: bob
       passwd: bob123

``codeDir`` 中的文件会自动上传到远程计算机中。 可在 Windows、Linux 或 macOS 上运行以下命令，在远程 Linux 计算机上启动 Trial：

.. code-block:: bash

   nnictl create --config examples/trials/mnist-annotation/config_remote.yml

配置 python 环境
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

默认情况下，命令和脚本将在远程计算机的默认环境中执行。 如果远程机器上有多个 python 虚拟环境，并且想在特定环境中运行实验，请使用 **preCommand** 来指定远程计算机上的 python 环境。 

以 ``examples/trials/mnist-tfv2`` 为例。 示例文件 ``examples/trials/mnist-tfv2/config_remote.yml`` 的内容如下：

.. code-block:: yaml

   authorName: default
   experimentName: example_mnist
   trialConcurrency: 1
   maxExecDuration: 1h
   maxTrialNum: 10
   #choice: local, remote, pai
   trainingServicePlatform: remote
   searchSpacePath: search_space.json
   #choice: true, false
   useAnnotation: false
   tuner:
     #choice: TPE, Random, Anneal, Evolution, BatchTuner, MetisTuner
     #SMAC (SMAC should be installed through nnictl)
     builtinTunerName: TPE
     classArgs:
       #choice: maximize, minimize
       optimize_mode: maximize
   trial:
     command: python3 mnist.py
     codeDir: .
     gpuNum: 0
   #machineList can be empty if the platform is local
   machineList:
     - ip: ${replace_to_your_remote_machine_ip}
       username: ${replace_to_your_remote_machine_username}
       sshKeyPath: ${replace_to_your_remote_machine_sshKeyPath}
       # Below is an example of specifying python environment.
       pythonPath: ${replace_to_python_environment_path_in_your_remote_machine}

远程计算机支持以重用模式运行实验。 在这种模式下，NNI 将重用远程机器任务来运行尽可能多的 Trial. 这样可以节省创建新作业的时间。 用户需要确保同一作业中的每个 Trial 相互独立，例如，要避免从之前的 Trial 中读取检查点。  
按照以下设置启用重用模式：

.. code-block:: yaml

   remoteConfig:
     reuse: true