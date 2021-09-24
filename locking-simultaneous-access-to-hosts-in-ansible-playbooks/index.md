# Locking Simultaneous Access to Hosts in Ansible Playbooks
## Allow tasks in a playbook to wait while other playbooks are running the same tasks on a host

Alexei.Karve

Tags: Cloud computing, Linux

Published on June 27, 2020 / Updated on July 13, 2020

### Overview

Skill Level: Any Skill Level

Ansible and bash

We need a solution to prevent multiple simultaneous playbook runs from running on same host. Only one playbook run should continue while others should wait. It should handle situations when playbook run fails to run completely or takes too long.

### Ingredients

Ansible, bash, python

### Step-by-step

#### 1. Introduction

Quite often we land up inadvertently running the same ansible playbook or multiple ansible playbooks simultaneously against the same host/endpoint. The modules may be idempotent but often cannot tolerate simultaneous execution and may result in failure. To prevent these unexpected failures due to concurrency, there is a need for locking access to the host and only allowing one of the playbooks to run at a time while the rest of the playbooks wait for the lock to be released.

This recipe will show the changes required to the playbooks with the multiple possibilities to implement the steps. The basic procedure is to check if a specific condition (lock) is present at startup. If it's locked, the tasks in the playbook wait. If it is not locked, a lock is created, and the tasks continue execution. At the end of the tasks from the playbook, the lock is deleted in order to allow the rest of the waiting playbooks runs to continue. The lock checking and creation can be done in the pre\_tasks that Ansible executes before executing any tasks mentioned in the playbook. The lock deletion can be done in the post\_tasks that Ansible executes after any tasks mentioned in the playbook are completed.

#### 2. Selecting a locking mechanism

A locking mechanism consists of two steps that need to be performed as an atomic operation: Checks for the existence of a lockfile and if no lockfile exists, create one and continue

Two different ways are shown to create this lock: First way to get this lock functionality using the file system is to create a lock directory (called lock\_dir below). This can be done either by running mkdir with the “command” module (approach 1a) or using the “file” module with “state: directory” (approach 1b). The second way is to use the ln command to create a lock file (approach 2). These approaches are explained in detail below.

**Approach 1a. Using the command module to create a directory**: The command mkdir "{{ lock\_dir }}" will result in an error if the lock\_dir is already present. This is handled by ignoring the errors for this task by setting the “ignore\_errors: true”. We still need to know if the directory was already present. The easiest way to achieve this is to register the return value (called lockcode) from the command module and check the return code (rc). If the return code is 1, it means the directory was already present. If it is 0, it means it was successfully created. The changed\_when is set to True when the lockcode.rc==0 and it is set to False when it is nonzero. The source code for the [test\_changed1.yaml](https://github.com/thinkahead/DeveloperRecipes/blob/master/LockingSimultaneousAccess/test_changed1.yaml "test_changed1.yaml") is shown below.

**test\_changed1.yaml**

```
---
- hosts: localhost
  connection: local
  gather_facts: no
  vars:
    lock_dir: "/tmp/ansible-playbook-{{ ansible_host }}.lock"
  tasks:
    - name: create lock dir if not exists
      command: mkdir "{{ lock_dir }}"
      args:
        warn: false
      register: lockcode
      ignore_errors: true
      changed_when: "lockcode.rc|int==0"
    - debug:
        msg: "{{ lockcode.changed }}"
```

Run this command before first execution to cleanup the lock\_dir:

```
rmdir /tmp/ansible-playbook-127.0.0.1.lock 2>/dev/null
```

First execution creates the lock\_dir and shows msg that changed is **_true_**

```
ansible-playbook test_changed1.yaml # First time
```

Output from the above run is as follows:

```
PLAY [localhost] ******************************************************************************************************************************************************

TASK [create lock dir if not exists] **********************************************************************************************************************************
changed: [localhost]

TASK [debug] **********************************************************************************************************************************************************
ok: [localhost] => {
 "msg": true
}
```

Run the same playbook again

```
ansible-playbook test_changed1.yaml # Second time
```

In this next run, the msg shows that change is **_false_** because the directory already exists. Note how the fatal error “File exists” is ignored.

```
PLAY [localhost] ******************************************************************************************************************************************************

TASK [create lock dir if not exists] **********************************************************************************************************************************
fatal: [localhost]: FAILED! => {"changed": false, "cmd": ["mkdir", "/tmp/ansible-playbook-127.0.0.1.lock"], "delta": "0:00:00.003816", "end": "2020-06-26 08:21:50.737262", "msg": "non-zero return code", "rc": 1, "start": "2020-06-26 08:21:50.733446", "stderr": "mkdir: /tmp/ansible-playbook-127.0.0.1.lock: File exists", "stderr_lines": ["mkdir: /tmp/ansible-playbook-127.0.0.1.lock: File exists"], "stdout": "", "stdout_lines": []}
...ignoring

TASK [debug] **********************************************************************************************************************************************************
ok: [localhost] => {
 "msg": false
}

PLAY RECAP ************************************************************************************************************************************************************
localhost : ok=2 changed=0 unreachable=0 failed=0 skipped=0 rescued=0 ignored=1
```

**Approach 1b. Using the file module to create a directory:** With the file module, you need to set two parameters: path (this is the absolute path of the directory) and state (set this as ‘directory’, the default value is ‘file’). If the playbook is executed again, since the directory is already present, nothing will be changed. The source code for the [test\_changed2.yaml](https://github.com/thinkahead/DeveloperRecipes/blob/master/LockingSimultaneousAccess/test_changed2.yaml "test_changed2.yaml") is shown below.

**test\_changed2.yaml**

```
---
- hosts: localhost
  connection: local
  gather_facts: no
  vars:
    lock_dir: "/tmp/ansible-playbook-{{ ansible_host }}.lock"
  tasks:
    - name: create lock_dir if not exists
      file:
        path: "{{ lock_dir }}"
        state: directory
      register: lockcode
    - debug:
        msg: "{{ lockcode.changed }}"
```

Run the same command as shown previously for first execution to cleanup the lock\_dir. Output from test\_change2.yaml is pretty much the same as for test\_changed1.yaml

First run creates the lock\_dir and the msg shows that changed is **_true._** Second run the msg shows that change is False because the directory already exists. There is no failed message because the file module is idempotent to make the directory present.

```
ansible-playbook test_changed2.yaml # First run  
ansible-playbook test_changed2.yaml # Second run
```

**Approach 2. Using the ln command to create a lock file:** Yet another way is to create a link to a lock file using the ln command. This has the added advantage of knowing which process has locked the file. This allows us to check if the process has already died leaving a bad lock file or killing the long running process if it has retained the lock beyond the allocated time.

```
echo "$$" > "{{ lock_file }}.$$" && ln "{{ lock_file }}.$$ "{{ lock_file }}"
```

In the above command, the $$ gives the pid of the process. Given the variable lock\_file, a file lock\_file.$$ is created with the pid extension and a hardlink of lock\_file is also created. Hard links act as a shortcut to that file that is hard linked. The inode of both is the same. A file in the file system is basically a link to an inode and a hard link then just creates another file with a link to the same underlying inode. When you delete a file, it removes one link to the underlying inode. The inode is only deleted when both links to the inode have been deleted. Source code for the [test\_changed3.yaml](https://github.com/thinkahead/DeveloperRecipes/blob/master/LockingSimultaneousAccess/test_changed3.yaml "test_changed3.yaml") is shown below.

**test\_changed3.yaml**

```
- hosts: localhost
  connection: local
  gather_facts: no
  vars:
    lock_file: "/tmp/ansible-playbook-{{ ansible_host }}.lock"
  tasks:
    - name: create lock_file if not exists
      shell: echo "$$" | tee "{{ lock_file }}.$$" && ln "{{ lock_file }}.$$" "{{ lock_file }}"
      args:
        warn: false
      register: lockcode
      ignore_errors: true
      changed_when: "lockcode.rc|int==0"
    - debug:
        msg: "{{ lockcode.changed }}"
    - name: delete lock_file.$$
      shell: cat "{{ lock_file }}";rm "{{ lock_file }}.{{ lockcode.stdout }}"
      args:
        warn: false
      ignore_errors: true
```

Run the [test\_changed3.yaml](https://github.com/thinkahead/DeveloperRecipes/blob/master/LockingSimultaneousAccess/test_changed3.yaml "test_changed3.yaml") first time.

```
ansible-playbook test_changed3.yaml -v # First run
```

On first run, the following test\_changed3.yaml playbook [creates a lock\_file.$$](https://github.com/thinkahead/DeveloperRecipes/blob/master/LockingSimultaneousAccess/test_changed3.yaml#L8 "creates a lock_file.$$") with the process id (38686) along with hard link to the lock\_file. Since it successfully creates the hard link, it [deletes the lock\_file.$$](https://github.com/thinkahead/DeveloperRecipes/blob/master/LockingSimultaneousAccess/test_changed3.yaml#L17 "deletes the lock_file.$$ ") with the process id (38686). The lock\_file (containing the contents 38686) is still present.

```
PLAY [localhost] ******************************************************************************************************************************************************

TASK [create lock_file if not exists] *********************************************************************************************************************************
changed: [localhost] => {"changed": true, "cmd": "echo \"$$\" | tee \"/tmp/ansible-playbook-127.0.0.1.lock.$$\" && ln \"/tmp/ansible-playbook-127.0.0.1.lock.$$\" \"/tmp/ansible-playbook-127.0.0.1.lock\"", "delta": "0:00:00.011746", "end": "2020-06-26 11:23:19.908523", "rc": 0, "start": "2020-06-26 11:23:19.896777", "stderr": "", "stderr_lines": [], "stdout": "38686", "stdout_lines": ["38686"]}

TASK [debug] **********************************************************************************************************************************************************
ok: [localhost] => {
 "msg": true
}

TASK [delete lock_file.$$] ********************************************************************************************************************************************
changed: [localhost] => {"changed": true, "cmd": "cat \"/tmp/ansible-playbook-127.0.0.1.lock\";rm \"/tmp/ansible-playbook-127.0.0.1.lock.38686\"", "delta": "0:00:00.009671", "end": "2020-06-26 11:23:20.957812", "rc": 0, "start": "2020-06-26 11:23:20.948141", "stderr": "", "stderr_lines": [], "stdout": "38686", "stdout_lines": ["38686"]}

PLAY RECAP ************************************************************************************************************************************************************
localhost : ok=3 changed=2 unreachable=0 failed=0 skipped=0 rescued=0 ignored=0
```

Run the test\_changed3.yaml playbook again:

```
ansible-playbook test_changed3.yaml -v # Second run
```

In this second run, it is not able to link to the lock\_file (containing the contents 38686) because it already exists. It creates and deletes the new lock\_file.$$ with the new pid (38770). It also returns the old pid (38686) that has locked the lock\_file.

```
PLAY [localhost] ******************************************************************************************************************************************************

TASK [create lock_file if not exists] *********************************************************************************************************************************
fatal: [localhost]: FAILED! => {"changed": false, "cmd": "echo \"$$\" | tee \"/tmp/ansible-playbook-127.0.0.1.lock.$$\" && ln \"/tmp/ansible-playbook-127.0.0.1.lock.$$\" \"/tmp/ansible-playbook-127.0.0.1.lock\"", "delta": "0:00:00.010578", "end": "2020-06-26 11:23:44.601430", "msg": "non-zero return code", "rc": 1, "start": "2020-06-26 11:23:44.590852", "stderr": "ln: /tmp/ansible-playbook-127.0.0.1.lock: File exists", "stderr_lines": ["ln: /tmp/ansible-playbook-127.0.0.1.lock: File exists"], "stdout": "38770", "stdout_lines": ["38770"]}
...ignoring

TASK [debug] **********************************************************************************************************************************************************
ok: [localhost] => {
 "msg": false
}

TASK [delete lock_file.$$] ********************************************************************************************************************************************
changed: [localhost] => {"changed": true, "cmd": "cat \"/tmp/ansible-playbook-127.0.0.1.lock\";rm \"/tmp/ansible-playbook-127.0.0.1.lock.38770\"", "delta": "0:00:00.009414", "end": "2020-06-26 11:23:45.611196", "rc": 0, "start": "2020-06-26 11:23:45.601782", "stderr": "", "stderr_lines": [], "stdout": "38686", "stdout_lines": ["38686"]}

PLAY RECAP ************************************************************************************************************************************************************
localhost : ok=3 changed=1 unreachable=0 failed=0 skipped=0 rescued=0 ignored=1
```

For simplicity, in the following steps, we will use the the approach 1a - mkdir with command module as shown in [test\_changed1.yaml](https://github.com/thinkahead/DeveloperRecipes/blob/master/LockingSimultaneousAccess/test_changed1.yaml "test_changed1.yaml")

#### 3. Getting the number of seconds since epoch

We need to compare the time when the lock\_dir was created with the current time on the host. An easy way to compare the time is with epoch. The Unix epoch is the number of seconds that have elapsed since January 1, 1970 (midnight UTC/GMT), not counting leap seconds; 'epoch' is often used as a synonym for Unix time.

Simple mechanism to get the current epoch using bash is to run the command “date +%s” and get the stdout. This returns the number of seconds since the epoch. On Linux, date +%s%3N will give the number of milliseconds since the epoch and date +%s%6N will give the number of microseconds since epoch while date +%s%N will give the number of nanoseconds since epoch. Incidentally echo $(($(date +%s%N)/1000000)) will give the number of milliseconds since the epoch. The %3N and %6N in the previous date commands give the 3 and 6 most significant digits of the nanoseconds.

```
- name: Shell to Get Epoch
     command: date +%s
     register: ansible_date_time_epoch
   - debug:
       msg: "{{ ansible_date_time_epoch.stdout }}"
```

Yet another way is to create a simple python module that returns the epoch.

```
- name: Module to Get Epoch
  get_epoch:
  register: ansible_date_time_epoch
- debug:
    msg: "{{ ansible_date_time_epoch.epoch }}"
```

Here is the source code for the [get\_epoch.py](https://github.com/thinkahead/DeveloperRecipes/blob/master/LockingSimultaneousAccess/library/get_epoch.py "get_epoch.py") module in python that is placed in the library directory in the folder where the playbook is executed. It uses the time.time() to get the epoch.

**library/get\_epoch.py**

```
#!/usr/bin/python3

DOCUMENTATION = '''
---
module: get_epoch

short_description: Get the epoch

'''
RETURN = '''
epoch:
    description: Epoch
    type: int
    returned: always
'''

import time
from ansible.module_utils.basic import AnsibleModule

def main():
    fields = {
    }
    module = AnsibleModule(argument_spec=fields)
    result=int(time.time())
    module.exit_json(changed=True, epoch=result)

if __name__ == '__main__':
    main()
```

#### 4. Getting the epoch time of the lock\_dir

The stat.mtime gives the number of seconds since epoch for a file or directory. The following example shows how to retrieve the epoh time (type float – see the msg in Sample output) when the folder lock\_dir was last modified (in this case it is same as the time when the directory was created because we do not modify it).

```
- name: Get info on lock_dir
  stat:
    path: "{{ lock_dir }}"
  register: lock_dir_cfg
- debug:
    msg: "{{ lock_dir_cfg.stat.mtime }}"
```

**Sample output**

```
TASK [Get info on lock_dir] *******************************************************************************************************************************************
ok: [localhost]

TASK [debug] **********************************************************************************************************************************************************
ok: [localhost] => {
 "msg": "1593179419.24"
}
```

#### 5. Handling stale locks

Normally we expect that when a playbook run completes, the post\_tasks will delete the lock\_dir. However, state locks may result when a playbook fails to delete the lock\_dir either because of failed playbook or the playbook runs too long (assume it crashed and will not complete). Reliably detecting and removing a "stale" lock is impossible. However, we still need to identify this situation with a timeout and allow the rest of the waiting playbook runs to continue. We do not want them to wait forever. If there are more than one playbook runs that were waiting, then one of them needs to proceed while rest will wait for the lock again.

The following snippet show retrieving the ansible\_date\_time\_epoch.epoch and the lock\_dir\_cfg.stat.mtime. The two times are compared. The wait\_for module waits for the lock\_file to be deleted or a stale\_timeout (of 180 seconds default used below when the lock\_dir will become stale). The when condition invokes the wait\_for module if the difference is > 0 and < state\_timeout. Having the ansible\_date\_time\_epoch.epoch|float < lock\_dir\_cfg.stat.mtime is an error condition. Having the (ansible\_date\_time\_epoch.epoch|float - lock\_dir\_cfg.stat.mtime) >= state\_timeout means that the lock is stale.

```
- block:
    - name: Module to Get Epoch
      get_epoch:
      register: ansible_date_time_epoch
    - debug:
        msg: "{{ ansible_date_time_epoch.epoch }}"

    - name: Get info on lock_dir
      stat:
        path: "{{ lock_dir }}"
      register: lock_dir_cfg

    - debug:
        msg: "{{ lock_dir_cfg.stat.mtime }}"

    - name: Debug date comparison
      debug:
        msg: "wait_for timeout={{ (state_timeout - (ansible_date_time_epoch.epoch|float - lock_dir_cfg.stat.mtime))|int }}"

    - name: wait for lock file to be absent
      wait_for:
        path: "{{ lock_dir }}"
        state: absent
        timeout: "{{ (state_timeout - (ansible_date_time_epoch.epoch|float - lock_dir_cfg.stat.mtime))|int }}"
      ignore_errors: true
      register: waitfor
      when:
        - (ansible_date_time_epoch.epoch|float - lock_dir_cfg.stat.mtime) < state_timeout
        - (ansible_date_time_epoch.epoch|float - lock_dir_cfg.stat.mtime) > 0
```

When a stale lock is detected, we need to delete the lock\_dir. There are however multiple playbook runs all timing out at same time and all of them attempting to delete the lock\_dir. There is a race condition from multiple waiting playbooks. As a simplistic solution, the following block sets the state of the lock\_dir to absent (allowing it to be deleted) and give some time (2 seconds) for the runs to settle when the stale lock condition is detected.

```
- block:
    - name: remove lock file lock_dir
      file:
        path: "{{ lock_dir }}"
        state: absent
    - name: Sleep for 2 seconds for all waiting processes to make the file absent - May need to increase this
      wait_for:
        delay: 2
        timeout: 0
  when: (waitfor.skipped is defined and waitfor.skipped) or (waitfor.state is defined and waitfor.state!="absent")
```

#### 6. Looping on blocks

We require a loop around the multiple tasks for handling stale locks. It is natural to expect that a block should be repeatable. In Ansible however, there is no support for loop/until on blocks. We need a workaround for this if we want to avoid using custom modules. A naive approach is looping over blocks by turning them into dynamic includes. The rescue block with recursive include can be used to work around this restriction. Although we should prefer the declarative approach, in this case being imperative helps.

The outermost block in [wait\_for\_lock.yaml](https://github.com/thinkahead/DeveloperRecipes/blob/master/LockingSimultaneousAccess/wait_for_lock.yaml "wait_for_lock.yaml") [creates a lock\_dir](https://github.com/thinkahead/DeveloperRecipes/blob/master/LockingSimultaneousAccess/wait_for_lock.yaml#L1-L13 "creates a lock_dir") if it does not exist. It uses the stat module to [get the epoch](https://github.com/thinkahead/DeveloperRecipes/blob/master/LockingSimultaneousAccess/wait_for_lock.yaml#L18-L21 "get the epoch") when the directory was created. If a new directory was created (lock acquired), then it proceeds out of the block and pre\_tasks is completed. If the directory already exists (lockcode.rc|int==1), then it waits either till the lock\_dir gets deleted (by some other playbook run) or a state\_timeout (lock becoming stale). In the latter case, it deletes the lock\_dir and forces a failure. That causes the [rescue section to include the wait\_for\_lock.yaml](https://github.com/thinkahead/DeveloperRecipes/blob/master/LockingSimultaneousAccess/wait_for_lock.yaml#L65-L68 "rescue section") thus creating a loop. This will go back to the beginning of the block and try to create a lock\_dir. Since there may be multiple playbooks/runs occurring simultaneously, once of them will be successful in creating a new directory and get out of the pre\_tasks while others will have to stay in the pre\_tasks loop.

Shown in the [wait\_for\_lock.yaml](https://github.com/thinkahead/DeveloperRecipes/blob/master/LockingSimultaneousAccess/wait_for_lock.yaml "wait_for_lock.yaml") for “name: create lock dir if not exists” are two options to check whether the lock\_dir was created or if it already existed. One is to use the returncode with ignore\_errors: true with the condition “when: lockcode.rc|int==1”. This however results in error message that is ignored. If you do not like to see ignored error messages, the other approach is to echo the return code without generating the error. Thus we do not need to ignore\_errors and can use the condition “when: lockcode.stdout|int==1”. The option of using the file module is also shown but commented with its corresponding condition “when: not lockcode.changed”. Any one of these will work.

**Source code for wait\_for\_lock.yaml**

```
- block:
    - name: create lock dir if not exists
      shell: mkdir "{{ wait_for_lock_dir }}";echo $?
      args:
        warn: false
      register: lockcode
      ignore_errors: true
      throttle: 1
    # This is a better option than above create lock dir if not exists
    #- name: create lock dir if not exists
    #  file:
    #    path: "{{ wait_for_lock_dir }}"
    #    state: directory
    #  register: lockcode
    - name: Get info on wait_for_lock_dir
      stat:
        path: "{{ wait_for_lock_dir }}"
      register: wait_for_lock_dir_cfg

    - block:
        - name: Module to Get Epoch
          get_epoch:
          register: ansible_date_time_epoch
        - name: Debug date comparison
          debug:
            msg: "{{ uniqueid }} wait_for timeout={{ (180 - (ansible_date_time_epoch.epoch|float - wait_for_lock_dir_cfg.stat.mtime))|int }}"
        - name: wait for lock file to be absent
          wait_for:
            path: "{{ wait_for_lock_dir }}"
            state: absent
            timeout: "{{ (180 - (ansible_date_time_epoch.epoch|float - wait_for_lock_dir_cfg.stat.mtime))|int }}" # Change this to larger value the time required for the role to execute
          ignore_errors: true
          register: waitfor
          when:
            - (ansible_date_time_epoch.epoch|float - wait_for_lock_dir_cfg.stat.mtime) < 180
            - (ansible_date_time_epoch.epoch|float - wait_for_lock_dir_cfg.stat.mtime) > 0
        #- name: debug waitfor
        #  debug:
        #    msg: "{{ waitfor }}"
        - block:
            - name: remove lock file wait_for_lock_dir
              file:
                path: "{{ wait_for_lock_dir }}"
                state: absent
            - name: Sleep for 2 seconds for all waiting processes to make the file absent - May need to increase this
              wait_for:
                delay: 2
                timeout: 0
          when: (waitfor.skipped is defined and waitfor.skipped) or (waitfor.state is defined and waitfor.state!="absent")
        - fail:
            msg: Create loop on block and run again, rescue failure
      #when: lockcode.rc|int==1
      when: lockcode.stdout|int==1
      #when: not lockcode.changed

    - name: create lock dir if not exists, if it got deleted by other waiting workers
      file:
        path: "{{ wait_for_lock_dir }}"
        state: directory
      register: check_if_changed
    - name: Check if the wait_for_lock_dir needed to be recreated
      debug:
        msg: "CHECK This means some worker deleted the lock file: {{ check_if_changed.changed }}, increase the delay after remove lock file wait_for_lock_dir above"
      when: check_if_changed.changed
    - assert:
        that:
          - not check_if_changed.changed

  rescue:
    - debug:
        msg: "Running again"
    - include_tasks: wait_for_lock.yaml

  tags:
    - always
```

#### 7. Linear vs Free Strategy Plugins

With the [linear](https://docs.ansible.com/ansible/latest/plugins/strategy/linear.html) strategy, Ansible is designed so that each task will be run on all hosts before continuing on to the next task using a default of 5 forks. If you have 3 tasks it will ensure task 1 runs on all your hosts first, then task 2 is run, then task 3 is run. This parallel execution of tasks however causes a problem when we run simultaneous playbook runs against the hosts in same inventory because while one ansible-playbook run may lock some subset of the hosts, another run may lock another subset. The tasks from one playbook run that locked a subset of hosts will be waiting for the subset of hosts locked in another playbook run and will not be able to proceed (until the stale lock timeout). Ansible allows several play-level keywords that affect play execution, one of them is [serial](https://docs.ansible.com/ansible/latest/user_guide/playbooks_strategies.html#using-keywords-to-control-execution). To avoid this deadlock, we can set the “serial: 1”. That means each playbook run will restrict the batch size for a play to 1.

```
- hosts: all
  gather_facts: no
  serial: 1
```

Another keywork that we used is the [throttle](https://docs.ansible.com/ansible/latest/user_guide/playbooks_strategies.html#using-keywords-to-control-execution) in the file wait\_for\_lock.yaml task name: “create lock dir if not exists”. This throttle is not strictly required because Linux handles simultaneous create directory requests. Also, when we start multiple separate ansible-playbook processes, throttle does not work across processes.

Another alternative to using the above “linear strategy with serial:1” is to change the playbook to use the [free](https://docs.ansible.com/ansible/latest/plugins/strategy/free.html) strategy, that allows each host to run until the end of the play as fast as it can.

```
- hosts: all
  gather_facts: no
  strategy: free
```

#### 8. Testing

We can run simultaneous playbook runs with a shell command as below. The testlock runs the [wait\_for\_lock.yaml](https://github.com/thinkahead/DeveloperRecipes/blob/master/LockingSimultaneousAccess/wait_for_lock.yaml "wait_for_lock.yaml") in [pre\_tasks](https://github.com/thinkahead/DeveloperRecipes/blob/master/LockingSimultaneousAccess/testlock.yaml#L11-L17 "pre_tasks") and removes the lock\_dir using the file module in [post\_tasks](https://github.com/thinkahead/DeveloperRecipes/blob/master/LockingSimultaneousAccess/testlock.yaml#L18-L24 "post_tasks"). The tasks can perform some useful work. For testing, it creates and deletes a test-lock directory. The CHECK should not show any failures. The uniqueid passed as extra-vars is only for debugging.

You can run on localhost by uncommenting the two localhost lines in testlock.yaml

```
for i in {1..10}; do ansible-playbook testlock.yaml -e uniqueid=$i& done | grep -A1 CHECK
```

You can run it against multiple hosts in the inventory by changing the hosts line to the group or hostnames in your inventory.

```
for i in {1..10}; do ansible-playbook -i inventory testlock.yaml -e uniqueid=$i& done | grep -A1 CHECK
```

The [testlock.yml](https://github.com/thinkahead/DeveloperRecipes/blob/master/LockingSimultaneousAccess/testlock.yaml "testlock.yml") playbook sets the wait\_for\_lock\_dir variable used by the included wait\_for\_lock.yaml. This test playbook only includes the wait\_for\_lock.yaml in the pre\_tasks with associated “clear lock” in post\_tasks because we wanted to lock the full playbook will all its task. Since this wait\_for\_lock\_dir is a parameter, you can essentially have multiple lock files that you can create around different blocks or tasks and delete the respective lock file with the module call as shown in post\_tasks. This allows multiple separate locks.

**Source code for testlock.yaml**

```
#- hosts: localhost
#  connection: local
- hosts: ec2
  gather_facts: no
  serial: 1

  vars:
    lock_dir: "/tmp/ansible-playbook-{{ ansible_host }}.lock"
    state_timeout: 180 # seconds
    uniqueid: 0

  pre_tasks:
    - name: wait for lock
      include_tasks: wait_for_lock.yaml
      vars:
        wait_for_lock_dir: "{{ lock_dir }}"
      tags:
        - always

  post_tasks:
    - name: clear lock
      file:
        path: "{{ lock_dir }}"
        state: absent
      tags:
        - always

  tasks:
    - name: Do something useful
      debug:
        msg: "Creating and deleting a test-lock directory"
    - name: CHECK Create test-lock dir. This should not fail
      command: mkdir "test-lock"
      args:
        warn: false
      tags:
        - always
    - name: CHECK Delete test-lock dir. This should not fail
      shell: rmdir "test-lock"
      args:
        warn: false
      tags:
        - always
```

#### 9. Conclusion

The provided recipe along with the sample code shows how you can lock sections of tasks to prevent them from running simultaneously on hosts. Using links as a lock or creating an additional file containing the pid of locking process when using the directory as a lock would allow checking whether the waiting process is still alive and taking desired action to either fail, kill the previous process or continue simultaneously. You could also maintain a queue of waiting processes as files using the create date to find which process arrived earlier in the lock directory. This is left as an exercise for the reader.

Hope you have enjoyed the article. Share your thoughts in the comments or engage in the conversation with me on Twitter @aakarve. I look forward to hearing about how you prevent simultaneous executions on each host endpoint and if you would like to see something covered in more detail.

#### 10. References

Lock your script (against parallel execution) [https://wiki.bash-hackers.org/howto/mutex](https://wiki.bash-hackers.org/howto/mutex "Lock your script (against parallel execution)")

Feature request for until loop on blocks or includes [https://github.com/ansible/ansible/issues/46203](https://github.com/ansible/ansible/issues/46203#issuecomment-556013701 "Wait until success")

Source code for above recipe [https://github.com/thinkahead/DeveloperRecipes/tree/master/LockingSimultaneousAccess](https://github.com/thinkahead/DeveloperRecipes/tree/master/LockingSimultaneousAccess "Locking Simultaneous Access")
