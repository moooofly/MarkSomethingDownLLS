# Harbor 之 Notary 和 Clair

> Notary 相关内容本文没有详细说明；

## 只言片语

- If **Notary** is enabled, `ui_url_protocol` has to be https.
- Harbor has integrated with **`Notary`** (for **Docker Content Trust**) and **`Clair`** (for **vulnerability scanning**). However, the default installation does not include **Notary** or **Clair** service.
- To install Harbor with **Notary** service, add a parameter when you run `install.sh`: `sudo ./install.sh --with-notary`
- To install Harbor with **Clair** service, add a parameter when you run `install.sh`: `sudo ./install.sh --with-clair`
- If you want to install both **Notary** and **Clair**, you must specify both parameters in the same command: `sudo ./install.sh --with-notary --with-clair`
- More information about **Notary and Docker Content Trust**, please refer to Docker's documentation: https://docs.docker.com/engine/security/trust/content_trust/
- For more information about **Clair**, please refer to Clair's documentation: https://coreos.com/clair/docs/2.0.1/


## Managing lifecycle of Harbor when it's installed with Notary and Clair

If you have installed **Notary** and **Clair**, you should include both components in the `docker-compose` and `prepare` commands:

```
$ sudo docker-compose -f ./docker-compose.yml -f ./docker-compose.notary.yml -f ./docker-compose.clair.yml down -v
$ vim harbor.cfg
$ sudo prepare --with-notary --with-clair
$ sudo docker-compose -f ./docker-compose.yml -f ./docker-compose.notary.yml -f ./docker-compose.clair.yml up -d
```


----------


## Clair

Automatic container vulnerability and security scanning for appc and Docker

### Clair 在 harbor.cfg 中的配置

```
# vi harbor.cfg
...
#The password of the Clair's postgres database, only effective when Harbor is deployed with Clair.
#Please update it before deployment, subsequent update will cause Clair's API server and Harbor unable to access Clair's database.
clair_db_password = password
...
```

### Clair 相关文档

- [Clair 2.0.1 Documentation](https://coreos.com/clair/docs/2.0.1/) -- 引导
- [Integrations](https://coreos.com/clair/docs/2.0.1/integrations.html) -- 列举了一些能够和 clair 进行集成的工具（项目）
- [Clair v1 API](https://coreos.com/clair/docs/2.0.1/api_v1.html) -- 集成需要
- [Notifications](https://coreos.com/clair/docs/2.0.1/notifications.html) -- to inform an endpoint that changes to tracked vulnerabilities have occurred.


### [github.com/coreos/clair](https://github.com/coreos/clair/)

`Clair` is an open source project for the **static analysis** of vulnerabilities in application containers (currently including [appc](https://github.com/appc/spec) and [docker](https://github.com/docker/docker/blob/master/image/spec/v1.2.md)).

1. In regular intervals, `Clair` ingests **vulnerability metadata** from a configured set of sources and stores it in the database.
2. Clients use the `Clair API` to **index** their container images; this creates a list of **features** present in the image and stores them in the database.
3. Clients use the `Clair API` to **query** the database for vulnerabilities of a particular image; **correlating** vulnerabilities and features is done for each request, avoiding the need to rescan images.
4. When updates to vulnerability metadata occur, a notification can be sent to alert systems that a change has occured.

Our goal is to enable a more transparent view of the security of container-based infrastructure. Thus, the project was named `Clair` after the French term which translates to *clear*, *bright*, *transparent*.

#### [Terminology](https://github.com/coreos/clair/blob/master/Documentation/terminology.md)

Container:

- **Container** - the execution of an image
- **Image** - a set of tarballs that contain the filesystem contents and run-time metadata of a container
- **Layer** - one of the tarballs used in the composition of an image, often expressed as a filesystem delta from another layer

Specific to `Clair`:

- **Ancestry** - the `Clair`-internal representation of an Image
- **Feature** - anything that when present in a filesystem could be an indication of a vulnerability (e.g. the presence of a **file** or an **installed software package**)
- **Feature Namespace (`featurens`)** - a context around features and vulnerabilities (e.g. an **operating system** or a **programming language**)
- **Vulnerability Source (`vulnsrc`)** - the component of `Clair` that tracks upstream vulnerability data and imports them into `Clair`'s database
- **Vulnerability Metadata Source (`vulnmdsrc`)** - the component of `Clair` that tracks upstream vulnerability metadata and associates them with vulnerabilities in `Clair`'s database

#### [Understanding drivers, their data sources, and creating your own](https://github.com/coreos/clair/blob/master/Documentation/drivers-and-data-sources.md)

`Clair` is organized into many different software components all of which are dynamically registered at compile time. All of these components can be found in the `ext/` directory.

Driver Types:

- **featurefmt** -> parses features of a particular format out of a layer
- **featurens** -> 	identifies whether a particular namespaces is applicable to a layer
- **imagefmt** -> determines the location of the root filesystem location for a layer
- **notification** -> implements the transport used to notify of vulnerability changes
- **versionfmt** -> parses and compares version strings
- **vulnmdsrc** -> fetches vulnerability metadata and appends them to vulnerabilities being processed
- **vulnsrc** -> fetches vulnerabilities for a set of namespaces

Data Sources for the built-in drivers:

- [Debian Security Bug Tracker](https://security-tracker.debian.org/tracker)
- [Ubuntu CVE Tracker](https://launchpad.net/ubuntu-cve-tracker)
- [Red Hat Security Data](https://www.redhat.com/security/data/metrics)
- [Oracle Linux Security Data](https://linux.oracle.com/security/)
- [Alpine SecDB](http://git.alpinelinux.org/cgit/alpine-secdb/)
- [NIST NVD](https://nvd.nist.gov)


### Vulnerability scanning via Clair

Static analysis of vulnerabilities is provided through open source project `Clair`. You can initiate scanning **on a particular image**, or **on all images** in Harbor. Additionally, you can also set a policy to **scan all the images at a specified time everyday**.

#### Vulnerability metadata

**Clair depends on the vulnerability metadata to complete the analysis process**. After the first initial installation, Clair will automatically start to update the metadata database from different vulnerability repositories. The updating process may take a while based on the data size and network connection. If the database has not been fully populated, there is a warning message at the footer of the repository datagrid view. 

![](https://raw.githubusercontent.com/vmware/harbor/b38ebd6b29b7d4926fe6e78d4c3dab0f5860dadc/docs/img/clair_not_ready.png)

The 'database not fully ready' warning message is also displayed in the 'Vulnerability' tab of 'Configuration' section under 'Administration' for your awareness. 

![](https://raw.githubusercontent.com/vmware/harbor/b38ebd6b29b7d4926fe6e78d4c3dab0f5860dadc/docs/img/clair_not_ready2.png)

Once the database is ready, an overall database updated timestamp will be shown in the 'Vulnerability' tab of 'Configuration' section under 'Administration'. 

![](https://raw.githubusercontent.com/vmware/harbor/b38ebd6b29b7d4926fe6e78d4c3dab0f5860dadc/docs/img/clair_ready.png)

#### Scanning an image

Enter your project and locate the specified repository. Expand the tag list via clicking the arrow icon on the left side. For each tag there will be an 'Vulnerability' column to display vulnerability scanning status and related information. You can click on the vertical ellipsis to open a popup menu and then click on 'Scan' to start the vulnerability analysis process. 

![](https://raw.githubusercontent.com/vmware/harbor/b38ebd6b29b7d4926fe6e78d4c3dab0f5860dadc/docs/img/scan_menu_item.png)

> NOTES: Only the users with '**Project Admin**' role have the privilege to launch the analysis process.

The analysis process may have the following status that are indicated in the 'Vulnerability' column:

- **Not Scanned**: The tag has never been scanned.
- **Queued**: The scanning task is scheduled but not executed yet.
- **Scanning**: The scanning process is in progress.
- **Error**: The scanning process failed to complete.
- **Complete**: The scanning process was successfully completed.

For the '**Not Scanned**' and '**Queued**' statuses, a text label with status information is shown. For the '**Scanning**', a progress bar will be displayed. If an error occurred, you can click on the '**View Log**' link to view the related logs. 

If the process was successfully completed, a result bar is created. The width of the different colored sections indicates the percentage of features with vulnerabilities for a particular severity level.

- **Red**: High level of vulnerabilities
- **Orange**: Medium level of vulnerabilities
- **Yellow**: Low level of vulnerabilities
- **Grey**: Unknown level of vulnerabilities
- **Green**: No vulnerabilities 

![](https://raw.githubusercontent.com/vmware/harbor/b38ebd6b29b7d4926fe6e78d4c3dab0f5860dadc/docs/img/bar_chart.png)

Move the cursor over the bar, a tooltip with summary report will be displayed. Besides showing the total number of features with vulnerabilities and the total number of features in the scanned image tag, the report also lists the counts of features with vulnerabilities of different severity levels. The completion time of the last analysis process is shown at the bottom of the tooltip. 

![](https://raw.githubusercontent.com/vmware/harbor/b38ebd6b29b7d4926fe6e78d4c3dab0f5860dadc/docs/img/summary_tooltip.png)

Click on the tag name link, the detail page will be opened. Besides the information about the tag, all the vulnerabilities found in the last analysis process will be listed with the related information. You can order or filter the list by columns. 

![](https://raw.githubusercontent.com/vmware/harbor/b38ebd6b29b7d4926fe6e78d4c3dab0f5860dadc/docs/img/tag_detail.png)

> NOTES: You can initiate the vulnerability analysis for a tag at anytime you want as long as the status is not '**Queued**' or '**Scanning**'.

#### Scanning all images

In the 'Vulnerability' tab of 'Configuration' section under 'Administration', click on the '**SCAN NOW**' button to start the analysis process for all the existing images.

> NOTES: The scanning process is executed via multiple concurrent asynchronous tasks. There is no guarantee on the order of scanning or the returned results. 

![](https://raw.githubusercontent.com/vmware/harbor/b38ebd6b29b7d4926fe6e78d4c3dab0f5860dadc/docs/img/scan_all.png)

To avoid frequently triggering the resource intensive scanning process, the availability of the button is restricted. It can be only triggered once in a predefined period. The next available time will be displayed besides the button. 

![](https://raw.githubusercontent.com/vmware/harbor/b38ebd6b29b7d4926fe6e78d4c3dab0f5860dadc/docs/img/scan_all2.png)

#### Scheduled Scan by Policy

You can set policies to control the vulnerability analysis process. Currently, two options are available:

- **None**: No policy is selected.
- **Daily**: Policy is activated daily. It means an analysis job is scheduled to be executed at the specified time everyday. The scheduled job will scan all the images in Harbor. 

![](https://raw.githubusercontent.com/vmware/harbor/b38ebd6b29b7d4926fe6e78d4c3dab0f5860dadc/docs/img/scan_policy.png)

> NOTES: Once the scheduled job is executed, the completion time of scanning all images will be updated accordingly. Please be aware that the completion time of the images may be different because the execution of analysis for each image may be carried out at different time.


### 启动漏洞扫描后

Database updated on "Vulnerability database might not be fully ready!" ==> Vulnerability -> Scan All -> SCAN NOW "Available after 2017/12/06 13:32"

上述内容是因为，Clair 需要下载很多数据，以及进行漏洞扫描花费的时间比较长；

```
root@docker-registry-stag:/opt/apps/harbor# top
top - 03:41:51 up 407 days, 20:46,  2 users,  load average: 3.81, 3.32, 1.98
Tasks: 176 total,   2 running, 174 sleeping,   0 stopped,   0 zombie
%Cpu(s): 51.3 us,  3.5 sy,  0.0 ni,  0.0 id,  0.0 wa,  0.0 hi,  4.5 si, 40.6 st
KiB Mem:   3854828 total,  3639516 used,   215312 free,   135148 buffers
KiB Swap:        0 total,        0 used,        0 free.  1042528 cached Mem

  PID USER      PR  NI    VIRT    RES    SHR S %CPU %MEM     TIME+ COMMAND
15455 root      20   0  578880 226536   5596 S 27.2  5.9   3:45.49 clair
15719 root      20   0  843492 730036   5312 R 25.5 18.9   1:58.72 bzr
15572 root      20   0  348280  21324   7536 S 17.2  0.6   1:45.93 harbor_ui
    3 root      20   0       0      0      0 S  5.0  0.0 560:28.30 ksoftirqd/0
    7 root      20   0       0      0      0 S  5.0  0.0 795:45.75 rcu_sched
15231 root      20   0  414808  17100   7852 S  5.0  0.4   0:30.67 registry
28160 root      20   0 3616664 299216   9380 S  4.0  7.8   3466:54 dockerd
    8 root      20   0       0      0      0 S  2.3  0.0 371:23.10 rcuos/0
14922 root      20   0  118396   2224   1484 S  1.7  0.1   0:07.98 docker-proxy
14982 root      20   0  555700   4880   1668 S  1.0  0.1   0:04.53 rsyslogd
15199 root      20   0  133500   4036   1392 S  1.0  0.1   0:03.57 docker-containe
  380 root      20   0   30184  15156   2608 S  0.7  0.4 507:51.46 node_exporter
15795 root      20   0  411336  20128   5884 S  0.7  0.5   0:05.61 harbor_jobservi
  207 root      20   0       0      0      0 S  0.3  0.0  17:18.33 jbd2/xvda1-8
 1096 root      20   0  123144   2008    504 S  0.3  0.1 344:43.52 falcon-agent
15160 999       20   0 1316180 476808   7664 S  0.3 12.4   0:22.70 mysqld
15512 root      20   0  199036   1968   1392 S  0.3  0.1   0:03.21 docker-containe
15779 root      20   0  199036   4020   1396 S  0.3  0.1   0:01.78 docker-containe
16904 root      20   0       0      0      0 S  0.3  0.0   0:00.05 kworker/u30:1
16934 root      20   0   23684   1704   1152 R  0.3  0.0   0:00.02 top
    1 root      20   0   33620   2208    704 S  0.0  0.1  72:33.83 init
    ...
```

可以看到，运行漏洞扫描需要耗费相当的 CPU 资源；


----------

## 基于 vagrant 环境进行验证

```
[#8#root@ubuntu-1604 /opt/apps/harbor]$docker-compose down
Stopping nginx              ... done
Stopping harbor-jobservice  ... done
Stopping harbor-ui          ... done
Stopping registry           ... done
Stopping harbor-db          ... done
Stopping harbor-adminserver ... done
Stopping harbor-log         ... done
Removing nginx              ... done
Removing harbor-jobservice  ... done
Removing harbor-ui          ... done
Removing registry           ... done
Removing harbor-db          ... done
Removing harbor-adminserver ... done
Removing harbor-log         ... done
Removing network harbor_harbor
[#9#root@ubuntu-1604 /opt/apps/harbor]$./install.sh --with-notary --with-clair

[Step 0]: checking installation environment ...

Note: docker version: 17.09.0

Note: docker-compose version: 1.16.1


[Step 1]: preparing environment ...
Clearing the configuration file: ./common/config/registry/root.crt
Clearing the configuration file: ./common/config/registry/config.yml
Clearing the configuration file: ./common/config/db/env
Clearing the configuration file: ./common/config/ui/app.conf
Clearing the configuration file: ./common/config/ui/private_key.pem
Clearing the configuration file: ./common/config/ui/env
Clearing the configuration file: ./common/config/jobservice/app.conf
Clearing the configuration file: ./common/config/jobservice/env
Clearing the configuration file: ./common/config/adminserver/env
Clearing the configuration file: ./common/config/nginx/cert/harbor_test.crt
Clearing the configuration file: ./common/config/nginx/cert/harbor_test.key
Clearing the configuration file: ./common/config/nginx/nginx.conf
loaded secret from file: /data/secretkey
Generated configuration file: ./common/config/nginx/nginx.conf
Generated configuration file: ./common/config/adminserver/env
Generated configuration file: ./common/config/ui/env
Generated configuration file: ./common/config/registry/config.yml
Generated configuration file: ./common/config/db/env
Generated configuration file: ./common/config/jobservice/env
Generated configuration file: ./common/config/jobservice/app.conf
Generated configuration file: ./common/config/ui/app.conf
Generated certificate, key file: ./common/config/ui/private_key.pem, cert file: ./common/config/registry/root.crt
Copying sql file for notary DB
Generated certificate, key file: ./cert_tmp/notary-signer-ca.key, cert file: ./cert_tmp/notary-signer-ca.crt
Generated certificate, key file: ./cert_tmp/notary-signer.key, cert file: ./cert_tmp/notary-signer.crt
Copying certs for notary signer
Copying notary signer configuration file
Generated configuration file: ./common/config/notary/server-config.json
Copying nginx configuration file for notary
Generated configuration file: ./common/config/nginx/conf.d/notary.server.conf
Generated and saved secret to file: /data/defaultalias
Generated configuration file: ./common/config/notary/signer_env
Generated configuration file: ./common/config/clair/postgres_env
Generated configuration file: ./common/config/clair/config.yaml
The configuration files are ready, please use docker-compose to start the service.


[Step 2]: checking existing instance of Harbor ...


[Step 3]: starting Harbor ...
Creating network "harbor_harbor" with the default driver
Creating network "harbor_notary-mdb" with the default driver
Creating network "harbor_notary-sig" with the default driver
Creating network "harbor_harbor-clair" with the default driver
Creating network "harbor_harbor-notary" with the default driver
Creating harbor-log ...
Creating harbor-log ... done
Creating clair-db ...
Creating notary-db ...
Creating clair-db
Creating registry ...
Creating harbor-db ...
Creating notary-db
Creating harbor-db
Creating registry
Creating harbor-adminserver ...
Creating clair-db ... done
Creating clair ...
Creating notary-db ... done
Creating notary-signer ...
Creating notary-signer ... done
Creating notary-server ...
Creating registry ... done
Creating harbor-ui ...
Creating harbor-ui ... done
Creating harbor-jobservice ...
Creating nginx ...
Creating nginx
Creating harbor-jobservice ... done

✔ ----Harbor has been installed and started successfully.----

Now you should be able to visit the admin portal at http://11.11.11.12.
For more details, please visit https://github.com/vmware/harbor .

[#10#root@ubuntu-1604 /opt/apps/harbor]$
```

各种情况下，命令输出结果有所不同

```
[#8#root@ubuntu-1604 /opt/apps/harbor]$ docker-compose -f ./docker-compose.yml -f ./docker-compose.clair.yml ps
       Name                     Command               State                                Ports
------------------------------------------------------------------------------------------------------------------------------
clair                /clair2.0.1/clair -config  ...   Up      6060/tcp, 6061/tcp
clair-db             /entrypoint.sh postgres          Up      5432/tcp
harbor-adminserver   /harbor/harbor_adminserver       Up
harbor-db            docker-entrypoint.sh mysqld      Up      3306/tcp
harbor-jobservice    /harbor/harbor_jobservice        Up
harbor-log           /bin/sh -c crond && rm -f  ...   Up      127.0.0.1:1514->514/tcp
harbor-ui            /harbor/harbor_ui                Up
nginx                nginx -g daemon off;             Up      0.0.0.0:443->443/tcp, 0.0.0.0:4443->4443/tcp, 0.0.0.0:80->80/tcp
registry             /entrypoint.sh serve /etc/ ...   Up      5000/tcp
[#9#root@ubuntu-1604 /opt/apps/harbor]$ docker-compose -f ./docker-compose.yml -f ./docker-compose.clair.yml -f ./docker-compose.notary.yml ps
       Name                     Command               State                                Ports
------------------------------------------------------------------------------------------------------------------------------
clair                /clair2.0.1/clair -config  ...   Up      6060/tcp, 6061/tcp
clair-db             /entrypoint.sh postgres          Up      5432/tcp
harbor-adminserver   /harbor/harbor_adminserver       Up
harbor-db            docker-entrypoint.sh mysqld      Up      3306/tcp
harbor-jobservice    /harbor/harbor_jobservice        Up
harbor-log           /bin/sh -c crond && rm -f  ...   Up      127.0.0.1:1514->514/tcp
harbor-ui            /harbor/harbor_ui                Up
nginx                nginx -g daemon off;             Up      0.0.0.0:443->443/tcp, 0.0.0.0:4443->4443/tcp, 0.0.0.0:80->80/tcp
notary-db            /docker-entrypoint.sh mysq ...   Up      3306/tcp
notary-server        /usr/bin/env sh -c /migrat ...   Up
notary-signer        /usr/bin/env sh -c /migrat ...   Up
registry             /entrypoint.sh serve /etc/ ...   Up      5000/tcp
[#10#root@ubuntu-1604 /opt/apps/harbor]$
[#10#root@ubuntu-1604 /opt/apps/harbor]$ docker-compose ps
       Name                     Command               State                                Ports
------------------------------------------------------------------------------------------------------------------------------
harbor-adminserver   /harbor/harbor_adminserver       Up
harbor-db            docker-entrypoint.sh mysqld      Up      3306/tcp
harbor-jobservice    /harbor/harbor_jobservice        Up
harbor-log           /bin/sh -c crond && rm -f  ...   Up      127.0.0.1:1514->514/tcp
harbor-ui            /harbor/harbor_ui                Up
nginx                nginx -g daemon off;             Up      0.0.0.0:443->443/tcp, 0.0.0.0:4443->4443/tcp, 0.0.0.0:80->80/tcp
registry             /entrypoint.sh serve /etc/ ...   Up      5000/tcp
[#11#root@ubuntu-1604 /opt/apps/harbor]$
```

在经过较长时间的 Clair 元数据更新后，效果如下

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/vuln%20-%201.png)

漏洞扫描示例

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/vuln%20-%202.png)

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/vuln%20-%203.png)

## 基于真实环境进行验证

> **建议针对有安全问题考量的镜像进行扫描查验**

- 当前是在之前的 harbor staging 所在机器（已废弃）上进行了 harbor 版本升级和 clair 功能使能；
- 升级后，该 harbor 能够访问到 **2017年11月30日** 前的镜像数据（即之前 harbor 进行版本升级时的内容）；

针对当前使用的 repo 进行扫描后的结果示例：

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/vuln%20-%2010.png)

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/vuln%20-%2012.png)

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/vuln%20-%2013.png)

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/vuln%20-%2014.png)

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/vuln%20-%2015.png)

更多扫描结果，可以通过 admin 账户登录到 https://10.1.1.37/harbor/sign-in 上进行查看；


------

补充：

- 使能 clair 功能后，会有一段较长的时间用于从外部数据源拉取安全元数据，在数据拉取完成前，虽然能够进行 Scan 动作，但结果不一定是准确的；
- Scan 行为会耗费一定的 CPU 资源；

