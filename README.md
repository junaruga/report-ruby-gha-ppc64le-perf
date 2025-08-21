# report-ruby-gha-ppc64le-perf

This repository is to manage logs related to the GitHub Actions ppc64lea
container image `ubuntu-24.04-ppc64le` which shows some timeout errors during
running unit tests by `make check`. I faced the issue on
[this PR ruby/ruby#14222](https://github.com/ruby/ruby/pull/14222).

## How I measured the performance of the `make check`

I added the following steps in the `.github/workflows/ubuntu.yml`.

The following step checks `cgroup` sys files. The output was saved as the
`sys_cgroup.log` in this repository.

```
      - name: Check cgroups sys files
        run: |
          set -x
          cat /sys/fs/cgroup/cpuset.cpus.effective || :
          sleep 0.1
          cat /sys/fs/cgroup/cpuset.cpus || :
          sleep 0.1
          cat /sys/fs/cgroup/cpu.max || :
          sleep 0.1
          cat /sys/fs/cgroup/cpu.stat || :
        working-directory:
```

The following step checks the availability of the used commands.

```
     - name: Check perf commands
        run: |
          command -v mpstat
          command -v vmstat
          command -v iostat
        working-directory:
```

The following step prints the outputs by the commands before running the unit
tests (`make check`). The output was saved as the `perf.log` in this repository.

```
      - name: Show perf values
        run: |
          set -x
          nproc
          uptime
          free -h
          df -hT
          df .
```

The following step is to run the `mpstat`, `vmstat` and `iostat` commands to
measure the unit test `make check`. The `30 10` means that the commands print
the outputs once in the 30 seconds for 10 times.

```
      - name: Set perf logs
        run: |
          set -x
          mkdir -p perf_logs
          mpstat -P ALL 30 10 > perf_logs/mpstat.log &
          vmstat 30 10 > perf_logs/vmstat.log &
          iostat -x 30 10 > perf_logs/iostat.log &
```

The following next step after the above step is essentially to execute the unit
tests, `make check`. If the CI case is ppc64le, forcibly return the exit-status
zero to proceed the steps.

```
      - name: make ${{ matrix.test_task }}
        run: |
          test -n "${LAUNCHABLE_STDOUT}" && exec 1> >(tee "${LAUNCHABLE_STDOUT}")
          test -n "${LAUNCHABLE_STDERR}" && exec 2> >(tee "${LAUNCHABLE_STDERR}")

          $SETARCH make -s ${{ matrix.test_task }} \
          ${TESTS:+TESTS="$TESTS"} \
          ${{ !contains(matrix.test_task, 'bundle') && 'RUBYOPT=-w' || '' }} \
          ${{ endsWith(matrix.os, 'ppc64le')  && '|| :' || '' }}
        timeout-minutes: ${{ matrix.timeout || 40 }}
```

The following next step after running the above step running unit tests
(`make check`) is to kill the `mpstat`, `vmstat` and `iostat` processes, and
print the content of the log files. The contents are saved as `mpstat.log`,
`vmstat.log` and `iostat.log` in this repository.

```
      - name: Show perf mpstat.log
        run: |
          pkill mpstat || :
          cat perf_logs/mpstat.log

      - name: Show perf vmstat.log
        run: |
          pkill vmstat || :
          cat perf_logs/vmstat.log

      - name: Show perf iostat.log
        run: |
          pkill iostat || :
          cat perf_logs/iostat.log
```

You can check my working branch
<https://github.com/junaruga/ruby/tree/wip/gha-add-ppc64le-report> and its
latest commit in my fork repository to reproduce this issue on your repository.

## CI logs

The log files in this repository were taken from the following CI log. This log
was used to interpret mainly.

https://github.com/ruby/ruby/actions/runs/17131140773/job/48595482790?pr=14222


This log was used to print the `cgroup` sys files later.

https://github.com/ruby/ruby/actions/runs/17134539800/job/48607085523?pr=14222


## Interpretation

### Running time of the `make check`

When checking the above CI log, the step `make check` is slow on the ppc64le.
The measured running times for the step for each CI image is below.

* ubuntu-24.04: 5m 23s
* ubuntu-24.04-arm: 3m 22s
* ubuntu-24.04-ppc64le: **24m 39s**
* ubuntu-24.04-s390x: 5m 20s

### Load average

The `report/ppc64le/perf.log` shows The number of the CPUs is 4 by the `nproc`.
So, the load average less than 4.0 is reasnable to run programs. However, the
output by the `uptime` shows that the load average is **34.58**, which is the
CPU is **8x over-used**. As a comparision, the following result isw with other
CPU logs. The s390x's load average **6.69** (> 4.0) is also a problem.

* `report/x86_64/perf.log`: CPU number: 4, load average: 3.30
* `report/arm64/perf.log`: CPU number: 4, load average: 2.53
* `report/ppc64le/perf.log`: CPU number: 4, load average: **34.58**
* `report/s390x/perf.log`: CPU number: 4, load average: **6.69**

### Something wrong setting for the exposed CPU numbers

The `report/ppc64le/mpstat.log` shows the `192 CPU` at the line 1. I think the
`192 CPU` is the host machine's CPU. However, I think you may need to limit the
number of the CPU as 4 in the container environment. The s390x's mpstat `8 CPU`
is also a problem.

* `report/x86_64/mpstat.log`: `nproc`: CPU number: 4, `mpstat`: `4 CPU`
* `report/arm64/mpstat.log`: `nproc` CPU number: 4, `mpstat`: `4 CPU`
* `report/ppc64le/mpstat.log`: `nproc` CPU number: 4, `mpstat`: **`192 CPU`**`
* `report/s390x/mpstat.log`: `nproc` CPU number: 4, `mpstat`: **`8 CPU`**

### cgroups sys files

According to the
[CI log](https://github.com/ruby/ruby/actions/runs/17134539800/job/48607085523?pr=14222)
with the "Check cgroups sys files" step, the content of the
`/sys/fs/cgroup/cpu.max` is the **`max 100000`** on the GitHub Actions images
`ubuntu-24.04-ppc64le` and `ubuntu-24.04-s390x` while the file is not found on
the GitHub Actions `ubuntu-24.04` and `ubuntu-24.04-arm`.

Maybe thre is a setting to limit the CPU max or not exposing this file in the
container.

* `report/x86_64/sys_cgroup.log`:
    ```
    + cat /sys/fs/cgroup/cpu.max
    cat: /sys/fs/cgroup/cpu.max: No such file or directory
    ```
* `report/arm64/sys_cgroup.log`:
    ```
    + cat /sys/fs/cgroup/cpu.max
    cat: /sys/fs/cgroup/cpu.max: No such file or directory
    ```
* `report/ppc64le/sys_cgroup.log`:
    ```
    + cat /sys/fs/cgroup/cpu.max
     max 100000
    ```
* `report/s390x/sys_cgroup.log`:
    ```
    + cat /sys/fs/cgroup/cpu.max
    max 100000
    ```

## Conclusion

The exposed high CPU numbers in the container enviornments coming from wrong
virtual machine settings in the GitHub Actions ppc64le and s390x may cause the
high load average.
