# Dell Latitude 5421 Ubuntu problems

## ADVANCED_SYSASSERT in iwlwifi

### Symptom

Wifi does not work.

### Description

This is a problem on a newer kernels. 

```sh
Processor model name: 11th Gen Intel(R) Core(TM) i7-11850H @ 2.50GHz
Network controller:   Intel Corporation Wi-Fi 6 AX210/AX211/AX411 160MHz (rev 1a)
```

Kernel logs

```sh
kernel: iwlwifi 0000:73:00.0: SecBoot CPU1 Status: 0x7663, CPU2 Status: 0xb03
kernel: iwlwifi 0000:73:00.0: UMAC PC: 0x8047f600
kernel: iwlwifi 0000:73:00.0: LMAC PC: 0x0
kernel: iwlwifi 0000:73:00.0: WRT: Collecting data: ini trigger 13 fired (delay=0ms).
kernel: iwlwifi 0000:73:00.0: Loaded firmware version: 66.f1c864e0.0 ty-a0-gf-a0-66.ucode
kernel: iwlwifi 0000:73:00.0: 0x00000000 | ADVANCED_SYSASSERT          
kernel: iwlwifi 0000:73:00.0: 0x00000000 | trm_hw_status0
kernel: iwlwifi 0000:73:00.0: 0x00000000 | trm_hw_status1
kernel: iwlwifi 0000:73:00.0: 0x00000000 | branchlink2
kernel: iwlwifi 0000:73:00.0: 0x00000000 | interruptlink1
kernel: iwlwifi 0000:73:00.0: 0x00000000 | interruptlink2
```

### Solution

It seems that pnvm doesn't match loaded microcode, pnvm file has to be renamed:

```sh
mv iwlwifi-ty-a0-gf-a0.pnvm iwlwifi-ty-a0-gf-a0.pnvm.bck
```

## Throtling problems

### Symptom

If system is under load for some time power throttling comes into play and
CPU frequency is capped at 800Mhz


- <https://github.com/whoenig/thinkpad-p14s-gen2-ubuntu>
- <https://github.com/erpalma/throttled/issues/255>
- <https://github.com/intel/thermal_daemon/issues/293>

## Tools

- [s-tui](https://github.com/amanusk/s-tui)

Installation

```sh
pip install s-tui --user
```

- [throttled](https://github.com/erpalma/throttled)

- script to restore power limits back to normal

Install cpupower

```sh
sudo apt-get install linux-tools-common
```

Script

```sh
#!/bin/sh

rmmod intel_rapl_msr
rmmod processor_thermal_device
rmmod processor_thermal_rapl
rmmod intel_rapl_common
rmmod intel_powerclamp

modprobe intel_powerclamp
modprobe intel_rapl_common
modprobe processor_thermal_rapl
modprobe processor_thermal_device
modprobe intel_rapl_msr

log=/var/log/cpu_fix.log
truncate -s 0 $log
printf "Log File - $(date)" >$log
while :; do
  PLUGGED=$(acpi -a | grep on-line)
  if [ ! -z "$PLUGGED" ]; then
    PERFORMANCE_GOVERNOR=$(cpupower frequency-info | grep "The governor \"performance\"")
    if [ -z "${PERFORMANCE_GOVERNOR}" ]; then
    echo "Setting governor to performance" >> $log
    /usr/bin/cpupower frequency-set -g performance >> $log 2>&1
    fi
    #echo "setting values" >>$log
    # MSR
    # PL1
    # ~75 watt
    echo 75000000 | tee /sys/devices/virtual/powercap/intel-rapl/intel-rapl:0/constraint_0_power_limit_uw > /dev/null 2>&1
    echo 28000000 | tee /sys/devices/virtual/powercap/intel-rapl/intel-rapl:0/constraint_0_time_window_us > /dev/null 2>&1
    # PL2
    echo 75000000 | tee /sys/devices/virtual/powercap/intel-rapl/intel-rapl:0/constraint_1_power_limit_uw > /dev/null 2>&1 # 44 watt
    echo 2440 | tee /sys/devices/virtual/powercap/intel-rapl/intel-rapl:0/constraint_1_time_window_us > /dev/null 2>&1    # 0.00244 sec

    # MCHBAR
    # PL1
    echo 75000000 | tee /sys/devices/virtual/powercap/intel-rapl-mmio/intel-rapl-mmio:0/constraint_0_power_limit_uw > /dev/null 2>&1 # 44 watt
    # ^ Only required change on a ASUS Zenbook UX430UNR
    echo 25000000 | tee /sys/devices/virtual/powercap/intel-rapl-mmio/intel-rapl-mmio:0/constraint_0_time_window_us > /dev/null 2>&1 # 28 sec
    # PL2
    echo 75000000 | tee /sys/devices/virtual/powercap/intel-rapl-mmio/intel-rapl-mmio:0/constraint_1_power_limit_uw > /dev/null 2>&1 # 44 watt
    echo 2440 | tee /sys/devices/virtual/powercap/intel-rapl-mmio/intel-rapl-mmio:0/constraint_1_time_window_us  >  /dev/null 2>&1   # 0.00244 secvim 
  fi
  sleep 5
done
```

Install and start as systemd service (placed `/etc/systemd/system/cpu-monitor.service`)

```ini
[Unit]
Description=CPU performance monitoring service for DELL laptops
Documentation=

[Service]
Type=simple
User=root
Group=root
TimeoutStartSec=0
Restart=on-failure
RestartSec=30s
#ExecStartPre=
ExecStart=/usr/local/share/systemd/services/cpufix.sh
SyslogIdentifier=CpuMonitor
#ExecStop=

[Install]
WantedBy=multi-user.target
```

Start and enable systemd service

```sh
$ systemctl start cpu-monitor.service 
$ systemctl enable cpu-monitor.service 
$ systemctl status cpu-monitor.service
```
