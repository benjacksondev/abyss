---
title: "Creating child processes in bash"
date: 2024-11-24
---

The following example can be used to create a Bash script that demonstrates forking and shows how to kill the parent process while also killing the child process.

## Example

```bash
#!/bin/bash

# Fork a true child process using a subshell
(
    # This is the child process
    trap "echo 'Child process $$ terminated'; exit" SIGTERM
    echo "Child process $$ started."
    while true; do
        echo "Child process $$ is running..."
        sleep 2
    done
) &

trap "echo 'Parent process $$ terminating...'; kill $!; wait $!; exit" SIGTERM

# Parent process
echo "Parent process $$ managing child process $!"
while true; do
    echo "Parent process $$ is running..."
    sleep 2
done
```

## Steps to Test

1. Save the script as `fork_example.sh` and make it executable:

    ```bash
    chmod u+x ./fork_example.sh
    ```

2. Run the script:

    ```bash
    ./fork_example.sh
    ```

3. Using another terminal, find the process IDs of the parent and child:

    ```bash
    ps aux | grep fork_example.sh
    ```

4. Kill the parent process:

    ```bash
    kill <parent_pid>
    ```

## Explanation

- `$$` is a special parameter for getting the PID of the script itself.
- `$!` is a special variable in Bash for getting the PID of the most recently started background process.
- The `trap` command in the child process ensures that it can clean up properly when receiving a `SIGTERM`.
- When the parent process is killed, the child process—being a direct descendant—is also terminated, since it has no parent to manage it.

This example demonstrates how process hierarchies in Unix-like systems work and how `trap` and `SIGTERM` can be used to clean up child processes.
