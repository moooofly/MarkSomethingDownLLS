# cadvisor 调试记录

将 cadvisor 二进制程序 scp 到 remote host 后，启动 cadvisor ，再执行 integration test 过程中抓取到的信息

```
root@harbor-service-new-stag:/opt/apps# ps auxf | grep cadvisor
root     16072  0.0  0.0  12920   932 pts/0    S+   08:07   0:00  |                   \_ grep --color=auto cadvisor
root     13666  0.0  0.0  11240  3080 ?        Ss   08:06   0:00      \_ bash -c sudo GORACE='halt_on_error=1' /tmp/cadvisor-26153/cadvisor --port 9100 --logtostderr --docker_env_metadata_whitelist=TEST_VAR  &> /tmp/cadvisor-26153/log.txt
root     13667  0.0  0.0  51428  3824 ?        S    08:06   0:00          \_ sudo GORACE=halt_on_error=1 /tmp/cadvisor-26153/cadvisor --port 9100 --logtostderr --docker_env_metadata_whitelist=TEST_VAR
root     13668  4.6  0.2 748808 43496 ?        Sl   08:06   0:01              \_ /tmp/cadvisor-26153/cadvisor --port 9100 --logtostderr --docker_env_metadata_whitelist=TEST_VAR
```


- The `GORACE` environment variable sets **race detector options**. The format is:

```
GORACE="option1=val1 option2=val2"
```

- `halt_on_error` (default 0): Controls whether the program exits after reporting first data race.



-----------------



测试命令

```
cd /go/src/github.com/google/cadvisor
./integration/runner/run.sh -port 9100 stag-reg.llsops.com dev.t.llsops.com
```

可以得到

- stderr 上输出的、本地跑 integration test 时的 glog 日志（当前程序也会直接打印从远端目标机器上 scp 回来的日志）
- 从目标机器上 scp 回来的 cAdvisor 运行时生成的 glog 日志
	- /tmp/cadvisor-xxxx/dev.t.llsops.com_log.txt
	- /tmp/cadvisor-xxxx/stag-reg.llsops.com_log.txt




```
+ '[' 5 == 0 ']'
+ go build -a github.com/google/cadvisor/integration/runner
+ HOSTS='-logtostderr -port 9100 stag-reg.llsops.com dev.t.llsops.com'
+ ./runner --logtostderr -logtostderr -port 9100 stag-reg.llsops.com dev.t.llsops.com
I0718 14:38:55.376458   18387 runner.go:239] Running integration tests on host(s) "stag-reg.llsops.com,dev.t.llsops.com"
I0718 14:38:55.377239   18387 runner.go:242] Building cAdvisor...(skipped by moooofly)
I0718 14:38:55.377389   18387 runner.go:94] Pushing cAdvisor binary to "stag-reg.llsops.com"...
I0718 14:38:55.377419   18387 runner.go:76] RunCommand => "ssh" ["stag-reg.llsops.com" "--" "mkdir" "-p" "/tmp/cadvisor-18387"]
I0718 14:38:55.378775   18387 runner.go:94] Pushing cAdvisor binary to "dev.t.llsops.com"...
I0718 14:38:55.378807   18387 runner.go:76] RunCommand => "ssh" ["dev.t.llsops.com" "--" "mkdir" "-p" "/tmp/cadvisor-18387"]
I0718 14:38:56.200011   18387 runner.go:76] RunCommand => "scp" ["-r" "cadvisor" "stag-reg.llsops.com:/tmp/cadvisor-18387"]
I0718 14:38:56.440390   18387 runner.go:76] RunCommand => "scp" ["-r" "cadvisor" "dev.t.llsops.com:/tmp/cadvisor-18387"]
I0718 14:39:14.163977   18387 runner.go:113] Running cAdvisor on "stag-reg.llsops.com"...
I0718 14:39:14.164162   18387 runner.go:76] RunCommand => "ssh" ["stag-reg.llsops.com" "--" "sudo GORACE='halt_on_error=1' /tmp/cadvisor-18387/cadvisor --port 9100 --logtostderr --docker_env_metadata_whitelist=TEST_VAR  &> /tmp/cadvisor-18387/log.txt"]
I0718 14:39:17.096955   18387 runner.go:113] Running cAdvisor on "dev.t.llsops.com"...
I0718 14:39:17.097018   18387 runner.go:76] RunCommand => "ssh" ["dev.t.llsops.com" "--" "sudo GORACE='halt_on_error=1' /tmp/cadvisor-18387/cadvisor --port 9100 --logtostderr --docker_env_metadata_whitelist=TEST_VAR  &> /tmp/cadvisor-18387/log.txt"]
I0718 14:39:17.154289   18387 runner.go:182] Running integration tests targeting "stag-reg.llsops.com"...
I0718 14:39:17.154311   18387 runner.go:76] RunCommand => "go" ["test" "--timeout" "15m0s" "github.com/google/cadvisor/integration/tests/..." "--host" "stag-reg.llsops.com" "--port" "9100" "--ssh-options" ""]
I0718 14:39:20.605055   18387 runner.go:182] Running integration tests targeting "dev.t.llsops.com"...
I0718 14:39:20.605082   18387 runner.go:76] RunCommand => "go" ["test" "--timeout" "15m0s" "github.com/google/cadvisor/integration/tests/..." "--host" "dev.t.llsops.com" "--port" "9100" "--ssh-options" ""]
I0718 14:40:01.326377   18387 runner.go:76] RunCommand => "scp" ["stag-reg.llsops.com:/tmp/cadvisor-18387/log.txt" "./"]
I0718 14:40:02.134894   18387 runner.go:218] ----------------------

Logs from Host: "stag-reg.llsops.com"
----------------------
I0718 06:39:14.985342   30135 storagedriver.go:50] Caching stats in memory for 2m0s
I0718 06:39:14.985566   30135 manager.go:155] cAdvisor running in container: "/sys/fs/cgroup/cpu,cpuacct/user.slice/user-0.slice/session-1707.scope"
I0718 06:39:15.423536   30135 fs.go:142] Filesystem UUIDs: map[164ecefb-ab3f-43e0-8418-0bc671bdac26:/dev/xvda1]
I0718 06:39:15.423558   30135 fs.go:143] Filesystem partitions: map[tmpfs:{mountpoint:/run major:0 minor:19 fsType:tmpfs blockSize:0} /dev/xvda1:{mountpoint:/var/lib/docker/overlay2 major:202 minor:1 fsType:ext4 blockSize:0} shm:{mountpoint:/var/lib/docker/containers/9a3cdcef11b22190b69558daae4b330c3c6ccd7b2f52ee57e748475586a2b5ee/shm major:0 minor:41 fsType:tmpfs blockSize:0}]
I0718 06:39:15.426614   30135 manager.go:229] Machine: {NumCores:8 CpuFrequency:2900058 MemoryCapacity:15768612864 HugePages:[{PageSize:1048576 NumPages:0} {PageSize:2048 NumPages:0}] MachineID:8476662e6b504629b4922f258921008e SystemUUID:EC27CBD3-D488-4BF7-BF92-346B9813EDCF BootID:0048a5ed-d805-4f37-a2ae-eaa357fdd073 Filesystems:[{Device:shm DeviceMajor:0 DeviceMinor:41 Capacity:67108864 Type:vfs Inodes:1924879 HasInodes:true} {Device:tmpfs DeviceMajor:0 DeviceMinor:19 Capacity:1576861696 Type:vfs Inodes:1924879 HasInodes:true} {Device:/dev/xvda1 DeviceMajor:202 DeviceMinor:1 Capacity:105546952704 Type:vfs Inodes:6553600 HasInodes:true}] DiskMap:map[202:0:{Name:xvda Major:202 Minor:0 Size:107374182400 Scheduler:none}] NetworkDevices:[{Name:br-608111f55282 MacAddress:02:42:23:8a:d8:8f Speed:0 Mtu:1500} {Name:ens3 MacAddress:02:a9:c2:e1:4e:42 Speed:10000 Mtu:9001}] Topology:[{Id:0 Memory:15768612864 Cores:[{Id:0 Threads:[0 4] Caches:[{Size:32768 Type:Data Level:1} {Size:32768 Type:Instruction Level:1} {Size:262144 Type:Unified Level:2}]} {Id:1 Threads:[1 5] Caches:[{Size:32768 Type:Data Level:1} {Size:32768 Type:Instruction Level:1} {Size:262144 Type:Unified Level:2}]} {Id:2 Threads:[2 6] Caches:[{Size:32768 Type:Data Level:1} {Size:32768 Type:Instruction Level:1} {Size:262144 Type:Unified Level:2}]} {Id:3 Threads:[3 7] Caches:[{Size:32768 Type:Data Level:1} {Size:32768 Type:Instruction Level:1} {Size:262144 Type:Unified Level:2}]}] Caches:[{Size:26214400 Type:Unified Level:3}]}] CloudProvider:AWS InstanceType:c4.2xlarge InstanceID:i-0d56f9198c187769d}
I0718 06:39:15.427773   30135 manager.go:235] Version: {KernelVersion:4.4.0-62-generic ContainerOsVersion:Ubuntu 16.04.1 LTS DockerVersion:17.09.0-ce DockerAPIVersion:1.32 CadvisorVersion:untagged-6617326b215cfd83af9b.14+01ef7f1fc34306-dirty CadvisorRevision:01ef7f1}
I0718 06:39:15.650906   30135 factory.go:356] Registering Docker factory
I0718 06:39:15.651254   30135 factory.go:143] Registering mesos factory
I0718 06:39:15.651267   30135 factory.go:54] Registering systemd factory
I0718 06:39:15.651475   30135 factory.go:97] Registering Raw factory
I0718 06:39:15.651689   30135 manager.go:1219] Started watching for new ooms in manager
I0718 06:39:15.652420   30135 manager.go:365] Starting recovery of all containers
I0718 06:39:15.744734   30135 manager.go:370] Recovery completed
I0718 06:39:15.789443   30135 cadvisor.go:172] Starting cAdvisor version: untagged-6617326b215cfd83af9b.14+01ef7f1fc34306-dirty-01ef7f1 on port 9100

----------------------
I0718 14:40:02.259757   18387 runner.go:76] RunCommand => "ssh" ["stag-reg.llsops.com" "--" "sudo" "pkill" "cadvisor"]
I0718 14:40:03.009250   18387 runner.go:76] RunCommand => "ssh" ["stag-reg.llsops.com" "--" "rm" "-rf" "/tmp/cadvisor-18387"]
I0718 14:40:06.227396   18387 runner.go:76] RunCommand => "scp" ["dev.t.llsops.com:/tmp/cadvisor-18387/log.txt" "./"]
I0718 14:40:07.158473   18387 runner.go:218] ----------------------

Logs from Host: "dev.t.llsops.com"
----------------------
I0718 06:39:19.436192   21131 storagedriver.go:50] Caching stats in memory for 2m0s
I0718 06:39:19.436543   21131 manager.go:155] cAdvisor running in container: "/sys/fs/cgroup/cpu,cpuacct/user.slice/user-0.slice/session-4326.scope"
I0718 06:39:19.530839   21131 fs.go:142] Filesystem UUIDs: map[164ecefb-ab3f-43e0-8418-0bc671bdac26:/dev/xvda1 d49b12ba-6ad6-4135-af2b-abeb21cc2cbd:/dev/xvdb]
I0718 06:39:19.530907   21131 fs.go:143] Filesystem partitions: map[tmpfs:{mountpoint:/run major:0 minor:18 fsType:tmpfs blockSize:0} /dev/xvda1:{mountpoint:/var/lib/docker/aufs major:202 minor:1 fsType:ext4 blockSize:0} /dev/xvdb:{mountpoint:/mnt major:202 minor:16 fsType:ext3 blockSize:0} shm:{mountpoint:/var/lib/docker/containers/fbf14db486ad5247f8b56e32d773ca3760c70c6087aeec0854843afb6ca39a4e/shm major:0 minor:43 fsType:tmpfs blockSize:0}]
I0718 06:39:19.534461   21131 manager.go:229] Machine: {NumCores:1 CpuFrequency:2500072 MemoryCapacity:3945181184 HugePages:[{PageSize:2048 NumPages:0}] MachineID:0eabb29c077e44779d6702a6fae42152 SystemUUID:EC262669-7526-125D-830E-47D285087C4C BootID:5259fe11-80c3-47b9-bfe5-121dd4e29efa Filesystems:[{Device:/dev/xvdb DeviceMajor:202 DeviceMinor:16 Capacity:4154654720 Type:vfs Inodes:262144 HasInodes:true} {Device:shm DeviceMajor:0 DeviceMinor:43 Capacity:67108864 Type:vfs Inodes:481589 HasInodes:true} {Device:tmpfs DeviceMajor:0 DeviceMinor:18 Capacity:394518528 Type:vfs Inodes:481589 HasInodes:true} {Device:/dev/xvda1 DeviceMajor:202 DeviceMinor:1 Capacity:52702224384 Type:vfs Inodes:3276800 HasInodes:true}] DiskMap:map[202:0:{Name:xvda Major:202 Minor:0 Size:53687091200 Scheduler:none} 202:16:{Name:xvdb Major:202 Minor:16 Size:4289200128 Scheduler:none}] NetworkDevices:[{Name:br-6f933938aab9 MacAddress:02:42:a1:2d:ef:0c Speed:0 Mtu:1500} {Name:eth0 MacAddress:02:03:ba:ff:29:34 Speed:0 Mtu:9001}] Topology:[{Id:0 Memory:3945181184 Cores:[{Id:0 Threads:[0] Caches:[{Size:32768 Type:Data Level:1} {Size:32768 Type:Instruction Level:1} {Size:262144 Type:Unified Level:2}]}] Caches:[{Size:26214400 Type:Unified Level:3}]}] CloudProvider:AWS InstanceType:m3.medium InstanceID:i-0c71d533d8849ffd1}
I0718 06:39:19.535180   21131 manager.go:235] Version: {KernelVersion:4.4.0-112-generic ContainerOsVersion:Ubuntu 16.04.1 LTS DockerVersion:1.13.1 DockerAPIVersion:1.26 CadvisorVersion:untagged-6617326b215cfd83af9b.14+01ef7f1fc34306-dirty CadvisorRevision:01ef7f1}
I0718 06:39:19.579341   21131 factory.go:356] Registering Docker factory
I0718 06:39:19.579806   21131 factory.go:143] Registering mesos factory
I0718 06:39:19.579826   21131 factory.go:54] Registering systemd factory
I0718 06:39:19.580037   21131 factory.go:97] Registering Raw factory
I0718 06:39:19.580249   21131 manager.go:1219] Started watching for new ooms in manager
I0718 06:39:19.581057   21131 manager.go:365] Starting recovery of all containers
I0718 06:39:19.884512   21131 manager.go:370] Recovery completed
I0718 06:39:20.091063   21131 cadvisor.go:172] Starting cAdvisor version: untagged-6617326b215cfd83af9b.14+01ef7f1fc34306-dirty-01ef7f1 on port 9100

----------------------
I0718 14:40:07.257046   18387 runner.go:76] RunCommand => "ssh" ["dev.t.llsops.com" "--" "sudo" "pkill" "cadvisor"]
I0718 14:40:08.046493   18387 runner.go:76] RunCommand => "ssh" ["dev.t.llsops.com" "--" "rm" "-rf" "/tmp/cadvisor-18387"]
I0718 14:40:09.663015   18387 runner.go:287] All tests pass!
I0718 14:40:09.663036   18387 runner.go:233] Execution time 1m14.286600588s
```


测试二：


```
go test --timeout 15m0s github.com/google/cadvisor/integration/tests/... --host stag-reg.llsops.com --port 9100 --ssh-options ""
```

得到

```
[#53#root@ubuntu-1604 /go/src/github.com/google/cadvisor]$go test --timeout 15m0s github.com/google/cadvisor/integration/tests/... --host stag-reg.llsops.com --port 9100 --ssh-options ""
--- FAIL: TestAttributeInformationIsReturned (10.01s)
	attributes_test.go:26: -------> TestAttributeInformationIsReturned
	attributes_test.go:33: unable to post "attributes" to "http://stag-reg.llsops.com:9100/api/v2.1/attributes": Get http://stag-reg.llsops.com:9100/api/v2.1/attributes: dial tcp: lookup stag-reg.llsops.com on 10.0.2.3:53: read udp 10.0.2.15:35999->10.0.2.3:53: i/o timeout
--- FAIL: TestDockerContainerById (12.65s)
	docker_test.go:74: -------> TestDockerContainerById
        Error Trace:    docker_test.go:63
	            	docker_test.go:82
	Error:      	Received unexpected error:
	            	unable to get "Docker container info for \"ef1bdc66650e8ee11b255c5ac8a7ecee2d73953915b021c8d6bf5a3b1293cd3d\"" from "http://stag-reg.llsops.com:9100/api/v1.3/docker/ef1bdc66650e8ee11b255c5ac8a7ecee2d73953915b021c8d6bf5a3b1293cd3d": Post http://stag-reg.llsops.com:9100/api/v1.3/docker/ef1bdc66650e8ee11b255c5ac8a7ecee2d73953915b021c8d6bf5a3b1293cd3d: dial tcp 10.1.0.13:9100: getsockopt: connection refused
	Messages:   	Timed out waiting for container "ef1bdc66650e8ee11b255c5ac8a7ecee2d73953915b021c8d6bf5a3b1293cd3d" to be available in cAdvisor: unable to get "Docker container info for \"ef1bdc66650e8ee11b255c5ac8a7ecee2d73953915b021c8d6bf5a3b1293cd3d\"" from "http://stag-reg.llsops.com:9100/api/v1.3/docker/ef1bdc66650e8ee11b255c5ac8a7ecee2d73953915b021c8d6bf5a3b1293cd3d": Post http://stag-reg.llsops.com:9100/api/v1.3/docker/ef1bdc66650e8ee11b255c5ac8a7ecee2d73953915b021c8d6bf5a3b1293cd3d: dial tcp 10.1.0.13:9100: getsockopt: connection refused
--- FAIL: TestDockerContainerByName (11.83s)
	docker_test.go:95: -------> TestDockerContainerByName
        Error Trace:    docker_test.go:63
	            	docker_test.go:107
	Error:      	Received unexpected error:
	            	unable to get "Docker container info for \"test-docker-container-by-name-10907\"" from "http://stag-reg.llsops.com:9100/api/v1.3/docker/test-docker-container-by-name-10907": Post http://stag-reg.llsops.com:9100/api/v1.3/docker/test-docker-container-by-name-10907: dial tcp 10.1.0.13:9100: getsockopt: connection refused
	Messages:   	Timed out waiting for container "test-docker-container-by-name-10907" to be available in cAdvisor: unable to get "Docker container info for \"test-docker-container-by-name-10907\"" from "http://stag-reg.llsops.com:9100/api/v1.3/docker/test-docker-container-by-name-10907": Post http://stag-reg.llsops.com:9100/api/v1.3/docker/test-docker-container-by-name-10907: dial tcp 10.1.0.13:9100: getsockopt: connection refused
--- FAIL: TestGetAllDockerContainers (13.42s)
	docker_test.go:133: -------> TestGetAllDockerContainers
        Error Trace:    docker_test.go:63
	            	docker_test.go:141
	Error:      	Received unexpected error:
	            	unable to get "Docker container info for \"a60f0ba7437f4563165f2cfcb7be02045cd3c21746225a07da5462fbc73dd897\"" from "http://stag-reg.llsops.com:9100/api/v1.3/docker/a60f0ba7437f4563165f2cfcb7be02045cd3c21746225a07da5462fbc73dd897": Post http://stag-reg.llsops.com:9100/api/v1.3/docker/a60f0ba7437f4563165f2cfcb7be02045cd3c21746225a07da5462fbc73dd897: dial tcp 10.1.0.13:9100: getsockopt: connection refused
	Messages:   	Timed out waiting for container "a60f0ba7437f4563165f2cfcb7be02045cd3c21746225a07da5462fbc73dd897" to be available in cAdvisor: unable to get "Docker container info for \"a60f0ba7437f4563165f2cfcb7be02045cd3c21746225a07da5462fbc73dd897\"" from "http://stag-reg.llsops.com:9100/api/v1.3/docker/a60f0ba7437f4563165f2cfcb7be02045cd3c21746225a07da5462fbc73dd897": Post http://stag-reg.llsops.com:9100/api/v1.3/docker/a60f0ba7437f4563165f2cfcb7be02045cd3c21746225a07da5462fbc73dd897: dial tcp 10.1.0.13:9100: getsockopt: connection refused
--- FAIL: TestBasicDockerContainer (11.79s)
	docker_test.go:159: -------> TestBasicDockerContainer
        Error Trace:    docker_test.go:63
	            	docker_test.go:173
	Error:      	Received unexpected error:
	            	unable to get "Docker container info for \"acc8f281310ed00146bc48fb4e0944f9eba04e912b0ef81cdc69420a2bcd4886\"" from "http://stag-reg.llsops.com:9100/api/v1.3/docker/acc8f281310ed00146bc48fb4e0944f9eba04e912b0ef81cdc69420a2bcd4886": Post http://stag-reg.llsops.com:9100/api/v1.3/docker/acc8f281310ed00146bc48fb4e0944f9eba04e912b0ef81cdc69420a2bcd4886: dial tcp: lookup stag-reg.llsops.com on 10.0.2.3:53: read udp 10.0.2.15:35223->10.0.2.3:53: i/o timeout
	Messages:   	Timed out waiting for container "acc8f281310ed00146bc48fb4e0944f9eba04e912b0ef81cdc69420a2bcd4886" to be available in cAdvisor: unable to get "Docker container info for \"acc8f281310ed00146bc48fb4e0944f9eba04e912b0ef81cdc69420a2bcd4886\"" from "http://stag-reg.llsops.com:9100/api/v1.3/docker/acc8f281310ed00146bc48fb4e0944f9eba04e912b0ef81cdc69420a2bcd4886": Post http://stag-reg.llsops.com:9100/api/v1.3/docker/acc8f281310ed00146bc48fb4e0944f9eba04e912b0ef81cdc69420a2bcd4886: dial tcp: lookup stag-reg.llsops.com on 10.0.2.3:53: read udp 10.0.2.15:35223->10.0.2.3:53: i/o timeout
--- FAIL: TestDockerContainerSpec (11.72s)
	docker_test.go:192: -------> TestDockerContainerSpec
        Error Trace:    docker_test.go:63
	            	docker_test.go:218
	Error:      	Received unexpected error:
	            	unable to get "Docker container info for \"38689239d07aa8497da742c473ca69812f8289bea567faec3e1a37ce9be36ed7\"" from "http://stag-reg.llsops.com:9100/api/v1.3/docker/38689239d07aa8497da742c473ca69812f8289bea567faec3e1a37ce9be36ed7": Post http://stag-reg.llsops.com:9100/api/v1.3/docker/38689239d07aa8497da742c473ca69812f8289bea567faec3e1a37ce9be36ed7: dial tcp: lookup stag-reg.llsops.com on 10.0.2.3:53: read udp 10.0.2.15:36840->10.0.2.3:53: i/o timeout
	Messages:   	Timed out waiting for container "38689239d07aa8497da742c473ca69812f8289bea567faec3e1a37ce9be36ed7" to be available in cAdvisor: unable to get "Docker container info for \"38689239d07aa8497da742c473ca69812f8289bea567faec3e1a37ce9be36ed7\"" from "http://stag-reg.llsops.com:9100/api/v1.3/docker/38689239d07aa8497da742c473ca69812f8289bea567faec3e1a37ce9be36ed7": Post http://stag-reg.llsops.com:9100/api/v1.3/docker/38689239d07aa8497da742c473ca69812f8289bea567faec3e1a37ce9be36ed7: dial tcp: lookup stag-reg.llsops.com on 10.0.2.3:53: read udp 10.0.2.15:36840->10.0.2.3:53: i/o timeout
--- FAIL: TestDockerContainerCpuStats (11.85s)
	docker_test.go:244: -------> TestDockerContainerCpuStats
        Error Trace:    docker_test.go:63
	            	docker_test.go:251
	Error:      	Received unexpected error:
	            	unable to get "Docker container info for \"62a1704cd0b2dfc1e19fa45a2f6e0e71bdd5c3260a784c8d5a4ab92384b7e1c2\"" from "http://stag-reg.llsops.com:9100/api/v1.3/docker/62a1704cd0b2dfc1e19fa45a2f6e0e71bdd5c3260a784c8d5a4ab92384b7e1c2": Post http://stag-reg.llsops.com:9100/api/v1.3/docker/62a1704cd0b2dfc1e19fa45a2f6e0e71bdd5c3260a784c8d5a4ab92384b7e1c2: dial tcp: lookup stag-reg.llsops.com on 10.0.2.3:53: read udp 10.0.2.15:49839->10.0.2.3:53: i/o timeout
	Messages:   	Timed out waiting for container "62a1704cd0b2dfc1e19fa45a2f6e0e71bdd5c3260a784c8d5a4ab92384b7e1c2" to be available in cAdvisor: unable to get "Docker container info for \"62a1704cd0b2dfc1e19fa45a2f6e0e71bdd5c3260a784c8d5a4ab92384b7e1c2\"" from "http://stag-reg.llsops.com:9100/api/v1.3/docker/62a1704cd0b2dfc1e19fa45a2f6e0e71bdd5c3260a784c8d5a4ab92384b7e1c2": Post http://stag-reg.llsops.com:9100/api/v1.3/docker/62a1704cd0b2dfc1e19fa45a2f6e0e71bdd5c3260a784c8d5a4ab92384b7e1c2: dial tcp: lookup stag-reg.llsops.com on 10.0.2.3:53: read udp 10.0.2.15:49839->10.0.2.3:53: i/o timeout
--- FAIL: TestDockerContainerMemoryStats (11.75s)
	docker_test.go:268: -------> TestDockerContainerMemoryStats
        Error Trace:    docker_test.go:63
	            	docker_test.go:275
	Error:      	Received unexpected error:
	            	unable to get "Docker container info for \"5f35605568eb87db9968a17318e42734ad730cc3cb9c8859ee82c4bbeb168d39\"" from "http://stag-reg.llsops.com:9100/api/v1.3/docker/5f35605568eb87db9968a17318e42734ad730cc3cb9c8859ee82c4bbeb168d39": Post http://stag-reg.llsops.com:9100/api/v1.3/docker/5f35605568eb87db9968a17318e42734ad730cc3cb9c8859ee82c4bbeb168d39: dial tcp: lookup stag-reg.llsops.com on 10.0.2.3:53: read udp 10.0.2.15:38998->10.0.2.3:53: i/o timeout
	Messages:   	Timed out waiting for container "5f35605568eb87db9968a17318e42734ad730cc3cb9c8859ee82c4bbeb168d39" to be available in cAdvisor: unable to get "Docker container info for \"5f35605568eb87db9968a17318e42734ad730cc3cb9c8859ee82c4bbeb168d39\"" from "http://stag-reg.llsops.com:9100/api/v1.3/docker/5f35605568eb87db9968a17318e42734ad730cc3cb9c8859ee82c4bbeb168d39": Post http://stag-reg.llsops.com:9100/api/v1.3/docker/5f35605568eb87db9968a17318e42734ad730cc3cb9c8859ee82c4bbeb168d39: dial tcp: lookup stag-reg.llsops.com on 10.0.2.3:53: read udp 10.0.2.15:38998->10.0.2.3:53: i/o timeout
--- FAIL: TestDockerContainerNetworkStats (11.86s)
	docker_test.go:290: -------> TestDockerContainerNetworkStats
        Error Trace:    docker_test.go:63
	            	docker_test.go:297
	Error:      	Received unexpected error:
	            	unable to get "Docker container info for \"20626175b41365d9fcb3287110ea202718a9deb533f69a32e8d264698f862bc3\"" from "http://stag-reg.llsops.com:9100/api/v1.3/docker/20626175b41365d9fcb3287110ea202718a9deb533f69a32e8d264698f862bc3": Post http://stag-reg.llsops.com:9100/api/v1.3/docker/20626175b41365d9fcb3287110ea202718a9deb533f69a32e8d264698f862bc3: dial tcp: lookup stag-reg.llsops.com on 10.0.2.3:53: read udp 10.0.2.15:56995->10.0.2.3:53: i/o timeout
	Messages:   	Timed out waiting for container "20626175b41365d9fcb3287110ea202718a9deb533f69a32e8d264698f862bc3" to be available in cAdvisor: unable to get "Docker container info for \"20626175b41365d9fcb3287110ea202718a9deb533f69a32e8d264698f862bc3\"" from "http://stag-reg.llsops.com:9100/api/v1.3/docker/20626175b41365d9fcb3287110ea202718a9deb533f69a32e8d264698f862bc3": Post http://stag-reg.llsops.com:9100/api/v1.3/docker/20626175b41365d9fcb3287110ea202718a9deb533f69a32e8d264698f862bc3: dial tcp: lookup stag-reg.llsops.com on 10.0.2.3:53: read udp 10.0.2.15:56995->10.0.2.3:53: i/o timeout
--- FAIL: TestDockerFilesystemStats (12.71s)
	docker_test.go:319: -------> TestDockerFilesystemStats
        Error Trace:    docker_test.go:63
	            	docker_test.go:343
	Error:      	Received unexpected error:
	            	unable to get "Docker container info for \"557d1cae4d157f71da54cba26463d0c6c942f537c1bee58b1d0441a11dae1965\"" from "http://stag-reg.llsops.com:9100/api/v1.3/docker/557d1cae4d157f71da54cba26463d0c6c942f537c1bee58b1d0441a11dae1965": Post http://stag-reg.llsops.com:9100/api/v1.3/docker/557d1cae4d157f71da54cba26463d0c6c942f537c1bee58b1d0441a11dae1965: dial tcp: lookup stag-reg.llsops.com on 10.0.2.3:53: read udp 10.0.2.15:35062->10.0.2.3:53: i/o timeout
	Messages:   	Timed out waiting for container "557d1cae4d157f71da54cba26463d0c6c942f537c1bee58b1d0441a11dae1965" to be available in cAdvisor: unable to get "Docker container info for \"557d1cae4d157f71da54cba26463d0c6c942f537c1bee58b1d0441a11dae1965\"" from "http://stag-reg.llsops.com:9100/api/v1.3/docker/557d1cae4d157f71da54cba26463d0c6c942f537c1bee58b1d0441a11dae1965": Post http://stag-reg.llsops.com:9100/api/v1.3/docker/557d1cae4d157f71da54cba26463d0c6c942f537c1bee58b1d0441a11dae1965: dial tcp: lookup stag-reg.llsops.com on 10.0.2.3:53: read udp 10.0.2.15:35062->10.0.2.3:53: i/o timeout
--- FAIL: TestMachineInformationIsReturned (10.08s)
	machine_test.go:24: -------> TestMachineInformationIsReturned
	machine_test.go:30: unable to get "machine info" from "http://stag-reg.llsops.com:9100/api/v1.3/machine": Get http://stag-reg.llsops.com:9100/api/v1.3/machine: dial tcp 10.1.0.13:9100: getsockopt: connection refused
--- FAIL: TestMachineStatsIsReturned (10.00s)
	machinestats_test.go:27: -------> TestMachineStatsIsReturned
	machinestats_test.go:33: unable to post "machine stats" to "http://stag-reg.llsops.com:9100/api/v2.1/machinestats": Get http://stag-reg.llsops.com:9100/api/v2.1/machinestats: dial tcp: lookup stag-reg.llsops.com on 10.0.2.3:53: read udp 10.0.2.15:44707->10.0.2.3:53: i/o timeout
FAIL
FAIL	github.com/google/cadvisor/integration/tests/api	139.660s
--- FAIL: TestHealthzOk (10.01s)
	healthz_test.go:26: ----> TestHealthzOk
	healthz_test.go:34: Get http://stag-reg.llsops.com:9100/healthz: dial tcp: lookup stag-reg.llsops.com on 10.0.2.3:53: read udp 10.0.2.15:37565->10.0.2.3:53: i/o timeout
FAIL
FAIL	github.com/google/cadvisor/integration/tests/healthz	10.017s
[#54#root@ubuntu-1604 /go/src/github.com/google/cadvisor]$
```



问题：

- rootfs 的存在与否代表什么含义？以及在 cadvisor 中出现的各个位置
- docker/rkt/CRI-O/runc 的关系





