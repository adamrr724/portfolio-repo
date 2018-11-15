---
layout: post
title:  "BLT Installation: Time saving tips & tricks"
date:   2018-11-14 15:35:28 -0800
categories: blt drupal installation
---

BLT is a complex tool Acquia Professional Services developers created to make building, testing, and spinning up new Drupal sites easier. It can be a very useful tool and make your life a lot easier -- if you can get it up and running.

In an effort to make this process easier for the average Joe or Jane trying to take advantage of the tools BLT has to offer, I have gathered a few of the common “gotchas” developers run into while trying to get BLT working on their projects.

These are specifically issues when Creating a BLT project from scratch.

### #1: BLT Bash profile not automatically installed

During the below BLT creation step (as outlined in the [BLT documentation here](https://blt.readthedocs.io/en/latest/creating-new-project/)):

```
composer create-project --no-interaction acquia/blt-project my-project
```

BLT should prompt you to add a BLT alias to your Bash Profile. Unfortunately, **sometimes** it doesn't prompt during the install, despite the following command being run (this ***should*** pop up a prompt asking if you'd like to install the blt shell alias):

```
blt:init:shell-alias
```

If you run into this issue, and the bash profile creation prompt is not triggered, you might run into the following error in the following steps:

```
blt: command not found
```

If that's the case, here are two simple commands you can run to fix it:

```
    composer run-script blt-alias
    source ~/.bash_profile
```

This will run BLT's shell alias script that should have originally on project creation -- and the second command refreshes your terminal session.

### #2: Running commands outside vs. inside the VM:
The BLT "Getting Started" documentation doesn't specify where each command needs to be run. Some commands, like `blt vm` (builds the VM instance), need to be run from the project directory.

Most, such as `blt setup`, `blt doctor`, or `blt blt drupal:install` are run from inside the VM.

If it's a command that needs to be run in the VM (like `blt setup`), then you must boot the VM, SSH into the VM, and run the BLT commands.

A demonstration of this is as follows (using Vagarant VM):

```
	vagrant up ### <-- Boot the VM
	vagrant ssh ### <-- SHH into the VM
	blt setup ### <-- Run your BLT Command
```
An example of this causing issues, is if you try to run `blt setup` from the project directory without SSHing into the VM.  

The confusing thing is  -- you ***will*** be able to run **some** of the commands, because BLT asks: ` Do you want to continue and execute this command on the host machine? (y/n)` -- meaning it will run these commands inside the VM.

This will work when running blt project commands, until it tries to reach something only currently running inside the VM (like Mysql -- example below):

```
> drupal:install
[warning] Drupal VM is locally initialized, but you are not inside the VM.
[warning] You should execute all BLT commands from within Drupal VM.
[warning] Use vagrant ssh to enter the VM.
 Do you want to continue and execute this command on the host machine? (y/n) y
[error]  MySql is not available. Please run `blt doctor` to diagnose the issue.
[error]  Command `drupal:install ` exited with code 1.
```

If you SSH into the VM and run the same command, everything runs smoothly and the `drupal:install` is successful.

### #3: VM Provisioning Issues

In certain situations you may spin up a VM and it could fail. This could happen for an assortment of reasons, but in my case -- it was because during the 10 minute install, my computer went to sleep (whoops!). If you try to run `blt vm` again, it can sometime fail with an error message about an existing VM. This is likely due to the `blt vm` command not making sure to delete both the vagrant instance and the VM provider instance, before attempting to recreate it.  

If this happens, you'll likely have to start from scratch. The easiest way to delete your vagrant instance is by ID.

Running the command `vagrant global-status` will provide the vagrant ID and VM provider:

```
id       name         provider   state   directory
----------------------------------------------------
8da772d  blt-project  virtualbox running /Users/adam.rossrussell/Desktop/blt-project
4d94978  new-project  virtualbox running /Users/adam.rossrussell/Desktop/new-project
f3e83db  blt-example1 virtualbox running /Users/adam.rossrussell/Desktop/blt-example1
```

To delete the Vagrant instance you would run the command `vagrant destroy ***ID***`.

If for some reason during setup the instance is rendered invalid, you may have to run the --prune flag `vagrant global-status -- prune` as well.

Now, if you are still running into existing VM errors after this step -- try deleting the provider instance as well.

If your provider is VirtualBox, you can list your provider instances by running `VBoxManage list vms` in the command line.

```
"local.blt-project.com" {9688cdfe-5bcd-443b-97b2-3bad7807e310}
"local.new-project.com" {d6c0925c-83f0-49bf-abeb-366f190d0dce}
"local.blt-example1.com" {25b5bea0-b4ff-4eb8-ac3c-586150e953d6}
```

The easiest way to delete these is by opening the VirtualBox GUI, right clicking on the VM instance, and pressing remove. You can either remove just the instance or delete all files associated with it.

![img](virtualbox GUI)


![img](delete options)

This should give you a blank slate to re-rerun `blt vm` -- and succesfully get the VM running.

### #4: Changing Configurations and the Default Tests:

There are many configurations or default tests you can set up via BLT.

To demonstrate how to change them, I'll use a particularly annoying configuration I ran into -- one that comes with BLT out of the box.

The git commit-msg pattern configuration (default in BLT) limits what you can write in commit messages -- without actually giving you direction on the validation the hook is looking for.

You might see an error like this:

```
Your local code has passed git pre-commit validation.
Executing .git/hooks/commit-msg...
Validating commit message syntax...

Warning: preg_match(): Empty regular expression in /Users/adam.rossrussell/Desktop/new-project/vendor/acquia/blt/src/Robo/Commands/Git/GitCommand.php on line 26
[error]  Invalid commit message!
Commit messages must conform to the regex
```

The BLT documentation to bypass the regex validated commit messages is not very clear -- but here is the fast, easy way to add a snippet to override that quickly and easily.

Add this snippet right underneath `git.remotes` configurations in the `blt/blt.yml`file.

```
  hooks:
      commit-msg: false
      pattern: false
```

The git configuration settings should now look like this:

```
git:
  default_branch: master
  remotes: { your remote URL }
  hooks:
      commit-msg: false
      pattern: false
```

Save changes. Next initiate the blt configuration changes by running `blt blt:init:git-hooks`. Now you should be able to run commit without running into errors.

*Side note: If you're feeling like extra credit, you can create your own git hooks -- instead of just turning them off!*

*Also, if you want to keep the commit message validation, but you're not sure what the regex is telling you -- the format is:*

`<prefix>-<ticket number> (e.g. NS-000): message of a certain length.`.

*This format is great when working off of tickets in Jira or Zendesk, but not very helpful for testing or personal development.*

Adding any configurations to the project are a similar endeavor.

Beware, there is a priority of configurations. The base config is found in the `build.yml` file (most general configurations) and will be overwritten by any configuration file higher than it in priority. `docroot/sites/[site]/[environment].blt.yml` is the most specific project base configuration, therefore any variable set in this file will overwrite any config lower in priority.

Config file priority (lowest priority to highest priority)

- `vendor/acquia/blt/config/build.yml`
- `blt/blt.yml`
- `blt/[environment].blt.yml`
- `docroot/sites/[site]/blt.yml`
- `docroot/sites/[site]/[environment].blt.yml`


#### Next installment (Coming soon)... Adding BLT to an existing Project.
