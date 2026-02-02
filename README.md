# lst-cluster-info

Practical guide for setting up and running Python / ML workloads on the LST HTCondor cluster at Saarland University.

This repository provides a step-by-step walkthrough for new users, covering SSH access, environment setup (Conda + pip), data transfer, and job submission via HTCondor.

## Audience

This guide is intended for students and researchers who:
- are new to remote cluster workflows
- want a minimal, practical path to running Python or ML jobs
- are working with GitHub-based projects on the LST cluster
  
## Getting started on the cluster

The [LST-Wiki](https://wiki2.coli.uni-saarland.de/doku.php?id=start) has a lot of good information, but it also has a lot of outdated information, and is lacking a lot of critical information for actually getting up and running on the cluster. If you have never set up remote server work before, this can all be really overwhelming and frustrating when it just doesn't seem to work. So I wanted to write down the steps I took to make this as easy as possible to start doing actual work.

First, you need an account. Apply for one [here](https://wiki2.coli.uni-saarland.de/doku.php?id=user:application_form)

## Connecting
Next, you need to [connect remotely](https://wiki2.coli.uni-saarland.de/doku.php?id=user:access) to the cluster. To do this, you'll want to configure your `ssh config` file. ([example](https://github.com/WilliamPLaCroix/LST-cluster-info/blob/835631ab93d0f9ba3220ef8f134b61b631f96de3/config)) Generally, in Windows machine, the SSH config file stored in the following location: `/c/Users/PC_USER_NAME/.ssh/` ([source](https://stackoverflow.com/questions/56287059/how-to-set-up-an-ssh-config-file-for-beginners)) On VSCode, this can be accessed by pressing `shift+ctrl+p` and selecting `Remote-SSH: Open SSH Configuration File`.

To avoid entering your password on each login, you can [configure SSH key-based authentication](https://superuser.com/questions/8077/how-do-i-set-up-ssh-so-i-dont-have-to-type-my-password). After this, you can log in by running `ssh submit` from your terminal.

## Home and Log Directories
If your home directory does not already exist, create it with: `mkdir /nethome/YOURUSERNEAME`. This will be your home base, but you still have the 100GB disk quota limit, so it is not where you want to store log files, model checkpoints, etc. Large files such as logs and model checkpoints should be stored in `/scratch/`, which provides effectively unlimited storage but is periodically cleaned. So make another directory via `mkdir /scratch/YOURUSERNAME` to store your log files, etc. 

Looking ahead, the [LST-Wiki example run.sub](https://wiki2.coli.uni-saarland.de/lib/exe/fetch.php?media=user:cluster:run_interactive.sub) points to log locations (`ouput, error, log`) in your scratch directory. We'll need to make those. `output` is where print statements and anything you see in a running terminal go, `error` is where your error traceback will end up, and `log` is a record of the job run with server-side info (probably not useful info for us). That directory structure is a little more complicated than I need for my current project, so I structured mine [like this](https://github.com/WilliamPLaCroix/LST-cluster-info/blob/main/run.sub). Either way, you will need to create those log directories in your scratch folder. From within `/scratch/YOURUSERNAME/`, create the following directory structure: enter `mkdir logs logs/err logs/out` (or whatever else you choose for your log directory structure).

With your file directories in place, let's upload some data! The easiest way to work back and forth between your local machine and the cluster is to push/pull everything through a [Github repo](https://github.com/). I'm assuming your project is already being stored on Github, if not, [here's a link to get you started](https://product.hubspot.com/blog/git-and-github-tutorial-for-beginners). Ensure your project is available in a GitHub repository before continuing. Make sure you're in `/nethome/YOURUSERNAME/` and `git clone YOURREPOSITORYURL`. 

## Conda
Next you'll need to setup your conda installation. **!NEW!**, this has to be done *while connected to the cluster node*, as just having a conda distro on `submit` doesn't allow you to run virtual environments from the actual compute node. At this point, you should be able to run commands with `HTCondor`. You can verify access by running `condor_q -all`, which lists all currently queued jobs. To [connect to a node](https://wiki2.coli.uni-saarland.de/doku.php?id=user:cluster:condor), you will need a [submit file](https://github.com/WilliamPLaCroix/LST-cluster-info/blob/4b74ec7056d456f318d47caa0fb9f6e1e71b4690/run.sub). Leave the docker/executable line commented out for now.

From your working directory, follow the instructions for submitting an interactive job, which will look like `condor_submit -i run.sub`, if you use the same lazy file names as I do.

Once this connects and finds a node, you can [install miniconda](https://docs.anaconda.com/miniconda/install/):

 - Enter `wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh`
 - Enter `bash Miniconda3-latest-Linux-x86_64.sh`
 - Type `yes` to agree to the license agreement.
 - Change your conda install location by typing `/nethome/YOURUSERNAME/miniconda3`
 - Type `yes` to activate conda on login (we will tinker with this later to be more streamlined)
 - The installer finishes and displays, “Thank you for installing Miniconda3!”
 - Enter `source ~/.bashrc`
 - You should see `(base)` in the command line prompt.
   
At this point, Miniconda is installed and available on the cluster node.

I like to just have one virtual environment that handles my whole project, so we're just going to [create one](https://docs.conda.io/projects/conda/en/latest/user-guide/tasks/manage-environments.html) and use it as our default, but `(base)` is always there if you need it.
Enter `conda create --name ENVIRONMENTNAME`
Type `y`
Enter `conda activate ENVIRONMENTNAME`
Now you should see `(ENVIRONMENTNAME)` before your directory in the shell.

**(OPTIONAL)** Before we start installing packages, if you want conda to activate the environment you just made automatically when you login, instead of typing `conda activate ENVIRONMENTNAME` every time, you can add it to your `bashrc` file by typing `vi ~/.bashrc`. This will open the `.bashrc` file (which runs automatically on login) in the editor. Press `i` to insert text, scroll down to the very bottom under the big `conda initialize` block, and add the line `conda activate ENVIRONMENTNAME`. Now you will automatically be in the environment on login (This does not affect your conda instance on the node, so we'll need to add it to our [shell script](https://github.com/WilliamPLaCroix/LST-cluster-info/blob/4b74ec7056d456f318d47caa0fb9f6e1e71b4690/run.sh) too, in a slightly different format)

## Pip and Libraries
Now you need to install the libraries for your project. Manually installing dependencies ad hoc via `pip install` is possible, but using a [requirements file](https://github.com/WilliamPLaCroix/LST-cluster-info/blob/4b74ec7056d456f318d47caa0fb9f6e1e71b4690/requirements.txt) is more reliable (this is just an example, do not use mine, your script won't run...). Again, you can make this yourself, or let python make it for you with [pipreqs](https://pypi.org/project/pipreqs/)! Go ahead and `pip install pipreqs`, then enter `pipreqs /LOCALPATHTOPROJECT`, and you should end up with a `requirements.txt` containing your project libraries (and versions). This will only list packages that were imported in `*.py` files within your project directory, so anything else you use will still need to be installed separately. Push this file to your repo. 

Back to the cluster.

If you're not still in the interactive session, then `ssh submit` and `condor_submit -i run.sub` your way back onto a node, we're not done yet. If the above steps worked, your terminal should look like `(ENVIRONMENTNAME) [YOURUSERNAME@SOME-NODE PROJECTDIRECTORY]$`. Great! Pull your requirements file onto the server. If you try to pip at this stage, it will probably give you an error since we didn't install pip yet... `conda install anaconda::pip` Now, from within your project directory, you can `pip install -r requirements.txt`, and it will collect and install all the packages you need to run your project! Once the dependencies are installed, the environment setup is complete.

## HTCondor
While you ***can*** just run your python scripts from this interactive session, they would really prefer you submit jobs through `HTCondor`, so that you're not just sitting and occupying a node unnecessarily. **IMPORTANT!**: Uncomment the executable line from your `run.sub` from earlier and push/pull your changes. `run.sub` just is there to establish your job's requirements and find you space on the cluster, but it does not run your script. That's where `run.sh` ([example](https://github.com/WilliamPLaCroix/LST-cluster-info/blob/4b74ec7056d456f318d47caa0fb9f6e1e71b4690/run.sh)) comes in, which primarily points to your python script, but can also do things like setting environment variables. I like to run a simple `helloworld.py` script ([example](https://github.com/WilliamPLaCroix/LST-cluster-info/blob/9f8a3e3db0354bbdfe4188e565e8e2b9257a0da7/helloworld.py)) to test some of my imports, and make sure my workflow is set up, before trying to run my project.

If all those pieces are in place, everything is installed and activated, then all you have to do now is enter `condor_submit run.sub`, and whatever script you pointed to in your run.sh file will get queued up and will (attempt to) run. `condor_q` to check if your job is running. Check your log files to verify that it ran smoothly. Your output file will not contain error statements if it didn't work, those will be stored in a separate error file. If the error file is empty, and the output file has everything you expected to see printed, then congratulations, you're successfully running jobs on the server!
