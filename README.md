
Assumptions
1. you have a running master node and compute node with rocky linux 8 installed on them

ssh to cluster, where C is your cluster number 

```
ssh root@hpcc-cluster-[C].stanford.edu
```

Following commands should be run as root user in the cluster

First we Install yum packages for headless rendering (Rendering graphics without a display, since we are on a remote server, this is the only way to run this repository)

The following command installs the packages in master node

```
yum -y install mesa-libglapi mesa-libOSMesa mesa-libOSMesa-devel mesa-libGL mesa-libGL-devel freeglut-devel mesa-libGLU mesa-libGLU-devel
```

Your terminal should output `Complete!` at the end if the installation was sucessful

You can check if the packages have been installed by running the following

```
rpm -q <package name>
```

It will output the complte file name of the package if the package is installed, otherwise it will just output "package not installed"

For instance 

```
rpm -q mesa-libOSMesa
>>> mesa-libOSMesa-22.3.0-2.el8.x86_64
```

The command below is to download the packages in the compute node. We install them to CHROOT a path defined during the creation of compute node which later becomes the root of compute node upon building. 

```
yum -y --installroot=$CHROOT install mesa-libglapi mesa-libOSMesa mesa-libOSMesa-devel mesa-libGL mesa-libGL-devel freeglut-devel mesa-libGLU mesa-libGLU-devel
```

Your terminal should output `Complete!` at the end if the installation was sucessful

The packages will only be installed after installing the new vnfs image and rebooting the compute node

Assembling the VNFS image. 
This command needs to be run each time we install a package to compute node, it usually takes a few minutes to run this command
<!-- Upload the path variable to warewolf database and build the compute image -->

```
wwvnfs --chroot=$CHROOT
```

This is a typical output you get


```

Using 'rocky8.8' as the VNFS name
Creating VNFS image from rocky8.8
Compiling hybridization link tree                           : 0.65 s
Building file list                                          : 2.47 s
Compiling and compressing VNFS                              : 35.97 s
Adding image to datastore                                   : 200.01 s
Total elapsed time                                          : 239.11 s

```

Now we have to reset the compute node so that the new packages get installed. The compute node uses the above generated VNFS image on next reboot

```
ipmitool -H 10.2.2.2 -U USERID -P PASSW0RD chassis power reset
```

Wait for 20 minutes before proceeding

run reboot, this will disconnect you from the cluster

```
reboot
```

It might a few minutes before you are able to reconnect

Now reconnect with your cluster

```
ssh root@hpcc-cluster-[C].stanford.edu
```


Now lets create new user. You can skip this step if you wish to use an existing user. Here we are creating a user named `student`. 

NOTE: the configuration files we will see later on assume that the user profile name is student. As will be pointed later on, you will need to make necessary changes if your user name is different than `student`

```
useradd -m student
```

Set the password for the new user

```
passwd student
```

The password files will get synced between master node and compute node in an interval but we can force the sync right away using the following command

```
wwsh file resync passwd shadow group
wwsh ssh compute-* /warewulf/bin/wwgetfiles
```

Switch to  `student` user and execute the following commands

NOTE All commands below should not be executed as `root` user. 

download conda. As per creation of this repo, we used the latest version available

```
wget https://repo.anaconda.com/archive/Anaconda3-2022.05-Linux-x86_64.sh
```

Make the file executable (this might require sudo priviledges so you may switch to root just for this specific command, but make sure it is executable by `student` user) 

```
chmod +x Anaconda3-2022.05-Linux-x86_64.sh
```

Run the file

```
./Anaconda3-2022.05-Linux-x86_64.sh
```

make sure you install conda inside the user 
so for student user. it should be installed in the path `/home/student/anaconda3` or  `/home/user-name/anaconda3` where user-name is the user-name of the linux user other than `root`

NOTE: As per the cluster where this program was built and tested, all user profiles resided in `/home/` directory

Initialise conda

```
conda init
``` 
or
```
conda init <shell-profile>
```


where `shell-profile` can be `bash` or `zsh` and so on. Check conda documentation for more details

if you encounter any errors ensure your path and bash file is correct and restart your terminal. 

Make sure to resolve any issues with initialising conda before proceeding. Refer Anaconda documentation (add link) for troubleshooting steps on installation 

Close and reopen your terminal for the changes to take effect

Now clone the following repo:

```
git clone https://github.com/Usgupta/AnimatedDrawings.git
```

This contains the complete program which we will run later. It is a forked repository of Meta's animated-drawings repo with some additional files added to ease the installation process on the cluster

Now change directory to the cloned repository

```
cd AnimatedDrawings
```

Create a conda environment using the environment file `environment.yml` present inside the repository


```
conda env create -f environment.yml
```

Activate the environment

```
conda activate animated-drawings
```

you should see animated-drawings beside your bash path, fot instance

```
(animated-drawings) [student@hpcc-cluster-21 AnimatedDrawings]$
```

Make sure the environment is activated before proceeding as it contains the necessary packages to run the program

Following commands enable using jupyter notebook inside the cluster

```
jupyter nbextension enable --py widgetsnbextension --user
jupyter nbextension enable nglview --py --user
```

You should see the following output for the respective commands

```
Enabling notebook extension jupyter-js-widgets/extension...
      - Validating: OK
Enabling notebook extension nglview-js-widgets/extension...
      - Validating: OK
```

Now lets create a slurm script which will create a jupyter notebook 

use `nano` or any other method to create a file named `create-jupyter-nb.slurm`.

We will use nano

```
nano create-jupyter-nb.slurm
```

Make sure your current user has the permissions to read write and execute this file

Add the following code to the file

```
#!/bin/bash
#SBATCH --job-name="Jupyter"                # Job name
#SBATCH --mail-user=[sunetid]@stanford.edu  # Email address    
#SBATCH --mail-type=NONE                    # Mail notification type (NONE, BEGIN, END, FAIL, ALL)
#SBATCH --partition=normal                  # Node partition
#SBATCH --nodes=1                           # Number of nodes requested
#SBATCH --ntasks=1                          # Number of processes
#SBATCH --time=01:00:00                     # Time limit request

conda init
conda activate ~/home/student/anaconda3/animated-drawings                       #Path of anaconda environment, change student to your user-profile name where the script is created
EXEC_DIR=$HOME/AnimatedDrawings
hostname && jupyter-notebook --no-browser --notebook-dir=$EXEC_DIR
```

Save the file and exit

Schedule the job using slurm

```
sbatch create-jupyter-nb.slurm
```

You should see an output showing that the job has been scheduled

```
Submitted batch job <job number>
```

If you found anything else, run sinfo to check compute node status and troubleshoot

Now let us get the jupyter notebook link, run the below command and check the output from the latest slurm out file 

```
egrep 'compute|localhost' slurm-*.out
```

You should an output with localhost link `http://localhost:8888/?token=`

Since the compute node generated the jupyter notebook, the localhost of compute node is not accessible by our computer. Hence we use the below command to map the port of the compute node to the port on our local machine


Open command prompt on your local machine (remember on the local machine and not in the cluster ) 

Run the following command 

```
ssh -L 8888:localhost:8888 student@hpcc-cluster-[C] -t ssh -L 8888:localhost:8888 compute-1-1
```

If you see any issues like port not found or port in use, it is likely a local machine issue, you may restart the computer or try killing the process which is using that port


Now you copy paste the link you got from the slurm script in the browser, you should be able to open jupyter

Create a new notebook and add the following code

```

from animated_drawings import render
render.start('examples/config/mvc/export_gif_example.yaml')
from IPython.display import Image
Image(open('video.gif','rb').read())
```

You should see the following output

```
Writing video to: /home/student/AnimatedDrawings/video.gif
100%|██████████| 339/339 [00:25<00:00, 13.21it/s]

```
<img src='./unused\ files/video.gif' width="200" height="200" /> </br></br></br>

Troubleshooting Steps:

AttributeError: 'GLXPlatform' object has no attribute 'OSMesa'

ssh to compute node, run python and set the PYOPENGL PLATFORM VARIABLE to osmesa. Below are the commands

```
ssh student@compute-1-1
run python
export PYOPENGL_PLATFORM=osmesa
```




