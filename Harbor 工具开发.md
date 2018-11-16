# Harbor 工具开发

TODO list:

- sync status analysis


----------


| policy ID | project (project ID) |
| -- | -- |
| 1 | backend (deleted) |
| 2 | backend-rls (5) |
| 3 | platform-rls (6) |
| 4 | library (1) |
| 5 | algorithm-rls (deleted) |
| 6 | algorithm-rls (10) |
| 7 | frontend-rls (14) |
| 8 | gcr-io-rls (16) |
| 9 | moooofly (deleted) |
| 10 | analysis (deleted) |
| 11 | analytics (deleted) |
| 12 | analytics-rls (37) |
| 13 | google_containers (46) |

## 相关 APIs

### 获取指定 policy (replication rule)

> 前提：需要事先知道 policy id 和 project id 的对应关系

```
[#1188#root@ubuntu-1604 /go/src/github.com/moooofly/harborctl]$./harborctl policy replication get -i 7
==> GET https://stag-reg.llsops.com/api/policies/replication/7
<==
<== Rsp Status: 200 OK
<== Rsp Body: {
  "id": 7,
  "project_id": 14,
  "target_id": 1,
  "name": "sync-to-prod",
  "enabled": 1,
  "description": "sync frontend release to prod harbor",
  "cron_str": "",
  "start_time": "2017-05-08T07:47:13Z",
  "creation_time": "2017-05-08T07:47:13Z",
  "update_time": "2017-05-08T07:47:13Z",
  "error_job_count": 0,
  "deleted": 0
}
[#1189#root@ubuntu-1604 /go/src/github.com/moooofly/harborctl]$
```

### 获取指定 project 信息

> 前提：需要事先知道 project id 对应的是哪个 project

```
[#1189#root@ubuntu-1604 /go/src/github.com/moooofly/harborctl]$./harborctl project get -j 14
==> GET https://stag-reg.llsops.com/api/projects/14
<==
<== Rsp Status: 200 OK
<== Rsp Body: {
  "project_id": 14,
  "owner_id": 1,
  "name": "frontend-rls",
  "creation_time": "2017-05-08T07:45:53Z",
  "creation_time_str": "",
  "deleted": 0,
  "owner_name": "",
  "public": 1,
  "Togglable": false,
  "update_time": "2017-05-08T07:45:53Z",
  "current_user_role_id": 0,
  "repo_count": 0,
  "enable_content_trust": false,
  "prevent_vulnerable_images_from_running": false,
  "prevent_vulnerable_images_from_running_severity": "",
  "automatically_scan_images_on_push": false
}
[#1190#root@ubuntu-1604 /go/src/github.com/moooofly/harborctl]$
```

### 查看基于指定 policy 的 jobs 发生的所有 error

> 前提：需要事先知道 policy id

```
[#1192#root@ubuntu-1604 /go/src/github.com/moooofly/harborctl]$./harborctl job replication list -h
This endpoint let user list jobs according to policy_id, and filters output by repository/status/start_time/end_time.

NOTE: if 'start_time' and 'end_time' are both null, list jobs of last 10 days

Usage:
  harborctl job replication list [flags]

Flags:
      --endTime string      The end time of jobs done to filter. (Format: yyyymmdd)
  -h, --help                help for list
  -n, --num int             The return list length number. (NOTE: not sure what it is for) (default 1)
  -p, --page int            The page nubmer, default is 1. (default 1)
  -s, --page_size int       The size of per page, default is 10, maximum is 100. (default 10)
  -i, --policy_id int       (REQUIRED) The ID of the policy that triggered this job.
  -r, --repository string   The returned jobs list filtered by repository name.
      --startTime string    The start time of jobs to filter. (Format: yyyymmdd)
      --status string       The returned jobs list filtered by status (one of [running|error|pending|retrying|stopped|finished|canceled])

Global Flags:
      --config string   config file (default is $HOME/, working dir (.), and ./conf dir)
[#1193#root@ubuntu-1604 /go/src/github.com/moooofly/harborctl]$
[#1193#root@ubuntu-1604 /go/src/github.com/moooofly/harborctl]$./harborctl job replication list -i 7 --status error
==> GET https://stag-reg.llsops.com/api/jobs/replication?policy_id=7&page=1&page_size=10&status=error&start_time=1540857600&end_time=1541721600&repository=&num=1
<==
<== Rsp Status: 200 OK
<== Rsp Body: [
  {
    "id": 20049,
    "status": "error",
    "repository": "frontend-rls/vira",
    "policy_id": 7,
    "operation": "transfer",
    "tags": [
      "v1.6.32-67284045"
    ],
    "creation_time": "2018-11-06T05:48:07Z",
    "update_time": "2018-11-06T05:48:07Z"
  },
  {
    "id": 19954,
    "status": "error",
    "repository": "frontend-rls/yy-admin",
    "policy_id": 7,
    "operation": "transfer",
    "tags": [
      "sandy-banner-5207e843"
    ],
    "creation_time": "2018-11-05T05:19:51Z",
    "update_time": "2018-11-05T05:19:51Z"
  },
  {
    "id": 19693,
    "status": "error",
    "repository": "frontend-rls/sprout-hybrid",
    "policy_id": 7,
    "operation": "transfer",
    "tags": [
      "v1.0.3-b3ce7f74"
    ],
    "creation_time": "2018-10-30T12:33:44Z",
    "update_time": "2018-10-30T12:33:45Z"
  },
  {
    "id": 19667,
    "status": "error",
    "repository": "frontend-rls/faas-node-runtime-latest",
    "policy_id": 7,
    "operation": "transfer",
    "tags": [
      "v0.2.0-13-7fbf6686"
    ],
    "creation_time": "2018-10-30T09:19:49Z",
    "update_time": "2018-10-30T09:19:50Z"
  },
  {
    "id": 19652,
    "status": "error",
    "repository": "frontend-rls/cc-hybrid-reborn",
    "policy_id": 7,
    "operation": "transfer",
    "tags": [
      "1913-172588__v6-8-6-bd0016ac"
    ],
    "creation_time": "2018-10-30T08:06:10Z",
    "update_time": "2018-10-30T08:06:10Z"
  }
]
[#1194#root@ubuntu-1604 /go/src/github.com/moooofly/harborctl]$
```

## 真实案例

> 以下内容基于 Harbor v1.2.2 版本

### 通过 CLI 查看 stag harbor 上的信息

![发生传输错误的一次同步](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/harbor%20-%20%E5%8F%91%E7%94%9F%E4%BC%A0%E8%BE%93%E9%94%99%E8%AF%AF%E7%9A%84%E4%B8%80%E6%AC%A1%E5%90%8C%E6%AD%A5.png)

从 harbor UI 上获取的 error 信息

```
2018-11-06T05:48:07Z [INFO] initializing: repository: frontend-rls/vira, tags: [v1.6.32-67284045], source URL: http://registry:5000, destination URL: https://prod-reg.llsops.com, insecure: false, destination user: admin
2018-11-06T05:48:07Z [INFO] initialization completed: project: frontend-rls, repository: frontend-rls/vira, tags: [v1.6.32-67284045], source URL: http://registry:5000, destination URL: https://prod-reg.llsops.com, insecure: false, destination user: admin
2018-11-06T05:48:07Z [WARNING] the status code is 409 when creating project frontend-rls on https://prod-reg.llsops.com with user admin, try to do next step
2018-11-06T05:48:07Z [ERROR] [transfer.go:351]: an error occurred while pulling manifest of frontend-rls/vira:v1.6.32-67284045 from http://registry:5000: 404 {"errors":[{"code":"MANIFEST_UNKNOWN","message":"manifest unknown","detail":{"Tag":"v1.6.32-67284045"}}]}
```

通过 `harborctl job replication list` 查看指定 repository 是否存在 replication 错误

> 可以发现，基于该命令查不到 Harbor UI 上给出的 error 详细信息

```
[#1194#root@ubuntu-1604 /go/src/github.com/moooofly/harborctl]$./harborctl job replication list -i 7 --status error -r frontend-rls/vira
==> GET https://stag-reg.llsops.com/api/jobs/replication?policy_id=7&page=1&page_size=10&status=error&start_time=1540857600&end_time=1541721600&repository=frontend-rls/vira&num=1
<==
<== Rsp Status: 200 OK
<== Rsp Body: [
  {
    "id": 20049,
    "status": "error",
    "repository": "frontend-rls/vira",
    "policy_id": 7,
    "operation": "transfer",
    "tags": [
      "v1.6.32-67284045"
    ],
    "creation_time": "2018-11-06T05:48:07Z",
    "update_time": "2018-11-06T05:48:07Z"
  }
]
[#1195#root@ubuntu-1604 /go/src/github.com/moooofly/harborctl]$
```

通过 `harborctl repository tag get` 确定指定 tag 是否存在

```
[#44#root@ubuntu-1604 /go/src/github.com/moooofly/harborctl]$./harborctl repository tag get -r frontend-rls/vira -t v1.6.32-67284045
==> GET https://stag-reg.llsops.com/api/repositories/frontend-rls/vira/tags/v1.6.32-67284045
<==
<== Rsp Status: 200 OK
<== Rsp Body: {
  "digest": "sha256:bf44a961a09617412312879726c0b48e343dee510e4be491e67b33d62531b8dc",
  "name": "v1.6.32-67284045",
  "architecture": "amd64",
  "os": "linux",
  "docker_version": "18.03.1-ce",
  "author": "",
  "created": "2018-11-06T05:48:05.991399477Z",
  "signature": null
}
[#45#root@ubuntu-1604 /go/src/github.com/moooofly/harborctl]$
```

通过 `harborctl repository tag manifest` 确定指定 tag 的 manifest 是否存在

> 从单纯确认 tag 是否存在的角度，可以不使用该 API

```
[#1195#root@ubuntu-1604 /go/src/github.com/moooofly/harborctl]$./harborctl repository tag manifest -r frontend-rls/vira -t v1.6.32-67284045 -v v2
==> GET https://stag-reg.llsops.com/api/repositories/frontend-rls/vira/tags/v1.6.32-67284045/manifest?version=v2
<==
<== Rsp Status: 200 OK
<== Rsp Body: {
  "manifest": {
    "schemaVersion": 2,
    "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
    "config": {
      "mediaType": "application/vnd.docker.container.image.v1+json",
      "size": 5773,
      "digest": "sha256:22b2a452cf12c385ac99bed647078c9f447fcc70e74ab6b0c5b36278c44a2d96"
    },
    "layers": [
      {
        "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
        "size": 23275620,
        "digest": "sha256:55c51e7de56129e402678fe7ba78d37f32af2da4315246061d63a03bb580e14f"
      },
      {
        "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
        "size": 21531803,
        "digest": "sha256:d0c913f7028b10df982ebcddfceca56bbabc74c1d19033b0b9c4fd7713162812"
      },
      {
        "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
        "size": 194,
        "digest": "sha256:c55908ce59ee805b7e31a2bbdf9fc3219e5cd6ff065fce4d19b11a50460f2792"
      },
      {
        "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
        "size": 463,
        "digest": "sha256:59f2e32312e028ca5db012b4a3f1bdebb91b18aded8c41622a71387520aed6fe"
      },
      {
        "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
        "size": 3243694,
        "digest": "sha256:eb71f9dcede9f65ecd0cfd84836e4ac614a9f22437f28facdf763bad3f4033a3"
      }
    ]
  },
  "config": "{\"architecture\":\"amd64\",\"config\":{\"Hostname\":\"77fbea4a3f5b\",\"Domainname\":\"\",\"User\":\"\",\"AttachStdin\":false,\"AttachStdout\":false,\"AttachStderr\":false,\"ExposedPorts\":{\"80/tcp\":{}},\"Tty\":false,\"OpenStdin\":false,\"StdinOnce\":false,\"Env\":[\"PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin\",\"NGINX_VERSION=1.13.0-1~stretch\",\"NJS_VERSION=1.13.0.0.1.10-1~stretch\",\"APP_HOME=/opt/apps/vira-hybrid\"],\"Cmd\":[\"nginx\",\"-g\",\"daemon off;\"],\"ArgsEscaped\":true,\"Image\":\"sha256:f8eedac9de03824c9f52fc8ce006272246d6a525e3c51fbefec22340bd413b83\",\"Volumes\":null,\"WorkingDir\":\"\",\"Entrypoint\":null,\"OnBuild\":[],\"Labels\":{\"maintainer\":\"Haohua Wu \\u003choward.wu@liulishuo.com\\u003e\"},\"StopSignal\":\"SIGQUIT\"},\"container_config\":{\"Hostname\":\"77fbea4a3f5b\",\"Domainname\":\"\",\"User\":\"\",\"AttachStdin\":false,\"AttachStdout\":false,\"AttachStderr\":false,\"ExposedPorts\":{\"80/tcp\":{}},\"Tty\":false,\"OpenStdin\":false,\"StdinOnce\":false,\"Env\":[\"PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin\",\"NGINX_VERSION=1.13.0-1~stretch\",\"NJS_VERSION=1.13.0.0.1.10-1~stretch\",\"APP_HOME=/opt/apps/vira-hybrid\"],\"Cmd\":[\"/bin/sh\",\"-c\",\"#(nop) COPY dir:c9c99296c32189ca536f644f7e5ad79c96635eed38368d81f7d4131728e020a9 in /opt/apps/vira-hybrid/ \"],\"ArgsEscaped\":true,\"Image\":\"sha256:f8eedac9de03824c9f52fc8ce006272246d6a525e3c51fbefec22340bd413b83\",\"Volumes\":null,\"WorkingDir\":\"\",\"Entrypoint\":null,\"OnBuild\":[],\"Labels\":{\"maintainer\":\"Haohua Wu \\u003choward.wu@liulishuo.com\\u003e\"},\"StopSignal\":\"SIGQUIT\"},\"created\":\"2018-11-06T05:48:05.991399477Z\",\"docker_version\":\"18.03.1-ce\",\"history\":[{\"created\":\"2017-05-08T23:36:50.013752134Z\",\"created_by\":\"/bin/sh -c #(nop) ADD file:a90ec883129f86b093f65b32e8e539168b462552a9fbf1c74d651a9bd9e9fc66 in / \"},{\"created\":\"2017-05-08T23:36:50.735498359Z\",\"created_by\":\"/bin/sh -c #(nop)  CMD [\\\"/bin/bash\\\"]\",\"empty_layer\":true},{\"created\":\"2017-05-10T13:43:02.339110159Z\",\"author\":\"NGINX Docker Maintainers \\\"docker-maint@nginx.com\\\"\",\"created_by\":\"/bin/sh -c #(nop)  MAINTAINER NGINX Docker Maintainers \\\"docker-maint@nginx.com\\\"\",\"empty_layer\":true},{\"created\":\"2017-05-10T13:43:03.039205706Z\",\"author\":\"NGINX Docker Maintainers \\\"docker-maint@nginx.com\\\"\",\"created_by\":\"/bin/sh -c #(nop)  ENV NGINX_VERSION=1.13.0-1~stretch\",\"empty_layer\":true},{\"created\":\"2017-05-10T13:43:03.788386786Z\",\"author\":\"NGINX Docker Maintainers \\\"docker-maint@nginx.com\\\"\",\"created_by\":\"/bin/sh -c #(nop)  ENV NJS_VERSION=1.13.0.0.1.10-1~stretch\",\"empty_layer\":true},{\"created\":\"2017-05-10T13:43:21.292659789Z\",\"author\":\"NGINX Docker Maintainers \\\"docker-maint@nginx.com\\\"\",\"created_by\":\"/bin/sh -c apt-get update \\t\\u0026\\u0026 apt-get install --no-install-recommends --no-install-suggests -y gnupg1 \\t\\u0026\\u0026 \\tNGINX_GPGKEY=573BFD6B3D8FBC641079A6ABABF5BD827BD9BF62; \\tfound=''; \\tfor server in \\t\\tha.pool.sks-keyservers.net \\t\\thkp://keyserver.ubuntu.com:80 \\t\\thkp://p80.pool.sks-keyservers.net:80 \\t\\tpgp.mit.edu \\t; do \\t\\techo \\\"Fetching GPG key $NGINX_GPGKEY from $server\\\"; \\t\\tapt-key adv --keyserver \\\"$server\\\" --keyserver-options timeout=10 --recv-keys \\\"$NGINX_GPGKEY\\\" \\u0026\\u0026 found=yes \\u0026\\u0026 break; \\tdone; \\ttest -z \\\"$found\\\" \\u0026\\u0026 echo \\u003e\\u00262 \\\"error: failed to fetch GPG key $NGINX_GPGKEY\\\" \\u0026\\u0026 exit 1; \\tapt-get remove --purge -y gnupg1 \\u0026\\u0026 apt-get -y --purge autoremove \\u0026\\u0026 rm -rf /var/lib/apt/lists/* \\t\\u0026\\u0026 echo \\\"deb http://nginx.org/packages/mainline/debian/ stretch nginx\\\" \\u003e\\u003e /etc/apt/sources.list \\t\\u0026\\u0026 apt-get update \\t\\u0026\\u0026 apt-get install --no-install-recommends --no-install-suggests -y \\t\\t\\t\\t\\t\\tnginx=${NGINX_VERSION} \\t\\t\\t\\t\\t\\tnginx-module-xslt=${NGINX_VERSION} \\t\\t\\t\\t\\t\\tnginx-module-geoip=${NGINX_VERSION} \\t\\t\\t\\t\\t\\tnginx-module-image-filter=${NGINX_VERSION} \\t\\t\\t\\t\\t\\tnginx-module-njs=${NJS_VERSION} \\t\\t\\t\\t\\t\\tgettext-base \\t\\u0026\\u0026 rm -rf /var/lib/apt/lists/*\"},{\"created\":\"2017-05-10T13:43:22.94801151Z\",\"author\":\"NGINX Docker Maintainers \\\"docker-maint@nginx.com\\\"\",\"created_by\":\"/bin/sh -c ln -sf /dev/stdout /var/log/nginx/access.log \\t\\u0026\\u0026 ln -sf /dev/stderr /var/log/nginx/error.log\"},{\"created\":\"2017-05-10T13:43:23.654985382Z\",\"author\":\"NGINX Docker Maintainers \\\"docker-maint@nginx.com\\\"\",\"created_by\":\"/bin/sh -c #(nop)  EXPOSE 80/tcp\",\"empty_layer\":true},{\"created\":\"2017-05-10T13:43:24.403276567Z\",\"author\":\"NGINX Docker Maintainers \\\"docker-maint@nginx.com\\\"\",\"created_by\":\"/bin/sh -c #(nop)  STOPSIGNAL [SIGQUIT]\",\"empty_layer\":true},{\"created\":\"2017-05-10T13:43:25.067075502Z\",\"author\":\"NGINX Docker Maintainers \\\"docker-maint@nginx.com\\\"\",\"created_by\":\"/bin/sh -c #(nop)  CMD [\\\"nginx\\\" \\\"-g\\\" \\\"daemon off;\\\"]\",\"empty_layer\":true},{\"created\":\"2018-11-06T05:48:05.314495868Z\",\"created_by\":\"/bin/sh -c #(nop)  LABEL maintainer=Haohua Wu \\u003choward.wu@liulishuo.com\\u003e\",\"empty_layer\":true},{\"created\":\"2018-11-06T05:48:05.502224682Z\",\"created_by\":\"/bin/sh -c #(nop)  ENV APP_HOME=/opt/apps/vira-hybrid\",\"empty_layer\":true},{\"created\":\"2018-11-06T05:48:05.72747077Z\",\"created_by\":\"/bin/sh -c #(nop) COPY file:dbbf819715d8569095138a1fd40cf0be7e831dd6a4b95c0a96cb571e245d86c2 in /etc/nginx/conf.d/ \"},{\"created\":\"2018-11-06T05:48:05.991399477Z\",\"created_by\":\"/bin/sh -c #(nop) COPY dir:c9c99296c32189ca536f644f7e5ad79c96635eed38368d81f7d4131728e020a9 in /opt/apps/vira-hybrid/ \"}],\"os\":\"linux\",\"rootfs\":{\"type\":\"layers\",\"diff_ids\":[\"sha256:8781ec54ba04ce83ebcdb5d0bf0b2bb643e1234a1c6c8bec65e8d4b20e58a90d\",\"sha256:f12c15fc56f1913acbb74fc732a2ae50e37d5a445f8d85ec5aa2dc755049219d\",\"sha256:08e6bf75740d026cea2ab8a543485f2d9254fdaaafcd39937c9f449dcbc28568\",\"sha256:f1f4c454dc711445692df3c7a01f959609b4f112a57830c0a7baf1af7edf6f32\",\"sha256:bfbc995cacdcfed0da891eae72986a3c88181abe752177a79bc36cc57f1f63f6\"]}}"
}
[#1196#root@ubuntu-1604 /go/src/github.com/moooofly/harborctl]$
```

### 通过 CLI 查看 prod harbor 上的信息

通过 `harborctl repository tag get` 确定指定 tag 是否存在

> 通过该 API 只能看到 tag 不存在

```
[#410#root@ubuntu-1604 /go/src/github.com/moooofly/harborctl]$./harborctl repository tag get -r frontend-rls/vira -t v1.6.32-67284045
==> GET https://prod-reg.llsops.com/api/repositories/frontend-rls/vira/tags/v1.6.32-67284045
<==
<== Rsp Status: 404 Not Found
<== Rsp Body: resource: frontend-rls/vira:v1.6.32-67284045 not found

[#411#root@ubuntu-1604 /go/src/github.com/moooofly/harborctl]$
```

通过 `harborctl repository tag manifest` 确定指定 tag 的 manifest 是否存在

> 通过该 API 能够看到 "MANIFEST_UNKNOWN" 这个信息

```
[#409#root@ubuntu-1604 /go/src/github.com/moooofly/harborctl]$./harborctl repository tag manifest -r frontend-rls/vira -t v1.6.32-67284045 -v v2
==> GET https://prod-reg.llsops.com/api/repositories/frontend-rls/vira/tags/v1.6.32-67284045/manifest?version=v2
<==
<== Rsp Status: 404 Not Found
<== Rsp Body: {"errors":[{"code":"MANIFEST_UNKNOWN","message":"manifest unknown","detail":{"Tag":"v1.6.32-67284045"}}]}

[#410#root@ubuntu-1604 /go/src/github.com/moooofly/harborctl]$
```

### 通过 CLI 触发 replication 的发生

> NOTE:
>
> - 经测试，该 API 在 Harbor v1.5.0-d59c257e 上有效
> - 下面的内容基于 v1.5.0-d59c257e 版本

通过 `harborctl policy replication list` 确定指定 policy 的详细信息

> 前提：需要事先知道 policy name

```
[#419#root@ubuntu-1604 /go/src/github.com/moooofly/harborctl]$./harborctl policy replication list -n sync-algorithm-rls
==> GET https://g-nprod-reg.llsops.com/api/policies/replication?name=sync-algorithm-rls&project_id=&page=1&page_size=10
<==
<== Rsp Status: 200 OK
<== Rsp Body: [
  {
    "id": 2,
    "name": "sync-algorithm-rls",
    "description": "sync algorithm-rls to global prod",
    "filters": null,
    "replicate_deletion": true,
    "trigger": {
      "kind": "Immediate",
      "schedule_param": null
    },
    "projects": [
      {
        "project_id": 6,
        "owner_id": 1,
        "name": "algorithm-rls",
        "creation_time": "2018-06-13T07:02:53Z",
        "update_time": "2018-06-13T07:02:53Z",
        "deleted": 0,
        "owner_name": "",
        "togglable": false,
        "current_user_role_id": 0,
        "repo_count": 0,
        "metadata": {
          "public": "true"
        }
      }
    ],
    "targets": [
      {
        "id": 1,
        "endpoint": "https://g-prod-reg.llsops.com",
        "name": "harbor-global-prod",
        "username": "admin",
        "password": "",
        "type": 0,
        "insecure": false,
        "creation_time": "2018-07-06T06:20:08Z",
        "update_time": "2018-07-06T06:20:08Z"
      }
    ],
    "creation_time": "2018-07-06T06:26:42Z",
    "update_time": "2018-07-09T11:30:45Z",
    "replicate_existing_image_now": false,
    "error_job_count": 2
  }
]
[#420#root@ubuntu-1604 /go/src/github.com/moooofly/harborctl]$
```

通过 `harborctl replication trigger` 触发复制

> 前提：需要实现知道 policy id

```
[#420#root@ubuntu-1604 /go/src/github.com/moooofly/harborctl]$./harborctl replication trigger -i 2
==> POST https://g-nprod-reg.llsops.com/api/replications
<==
<== Rsp Status: 200 OK
<== Rsp Body:
[#421#root@ubuntu-1604 /go/src/github.com/moooofly/harborctl]$
[#421#root@ubuntu-1604 /go/src/github.com/moooofly/harborctl]$date
Fri Nov  9 17:52:05 CST 2018
[#422#root@ubuntu-1604 /go/src/github.com/moooofly/harborctl]$
```

成功触发复制后

![在 policy level 上触发复制](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/harbor%20-%20%E5%9C%A8%20policy%20level%20%E4%B8%8A%E8%A7%A6%E5%8F%91%E5%A4%8D%E5%88%B6.png)

在 Harbor UI 上获取的其中一个 repository 的成功复制日志

```
2018-11-09T09:52:03Z [INFO] initialization completed: repository: algorithm-rls/lqpt, tags: [lqsrv-0.8.6-prod lqsrv-0.8.84 lqsrv-0.8.9-dev lqsrv-0.9.0-dev lqsrv-0.9.1-dev lqsrv-0.9.1-prod lqsrv-0.9.2-dev lqsrv-0.9.2-prod lqsrv-0.9.3-prod lqsrv-0.9.4-prod lqsrv-0.9.5-dev lqsrv-0.9.6-dev lqsrv-0.9.7-dev lqsrv-0.9.7-prod lqsrv-1.0.1-dev lqsrv-1.0.2-dev lqsrv-1.0.2-prod lqsrv-1.0.3-dev lqsrv-1.0.3-prod lqsrv-1.0.4-dev lqsrv-1.0.4-prod lqsrv_global-0.7.2], source registry: URL-https://g-nprod-reg.llsops.com insecure-true, destination registry: URL-https://g-prod-reg.llsops.com insecure-false
2018-11-09T09:52:03Z [WARNING] the status code is 409 when creating project algorithm-rls on destination registry, try to do next step
2018-11-09T09:52:03Z [INFO] manifest of algorithm-rls/lqpt:lqsrv-0.8.6-prod pulled successfully from source registry: sha256:31d8ec0fa4104732db921ac56bee918d204ba42ea53edd0c2f641358795fd3e7
2018-11-09T09:52:03Z [INFO] blob sha256:5590f6061554e38c3ce415c8448d3f2b5b57b789750adf891ccf49fd6d63cfb2 of algorithm-rls/lqpt:lqsrv-0.8.6-prod already exists on the destination registry, skip
2018-11-09T09:52:03Z [INFO] blob sha256:1e76f742da490c8d7c921e811e5233def206e76683ee28d735397ec2231f131d of algorithm-rls/lqpt:lqsrv-0.8.6-prod already exists on the destination registry, skip
2018-11-09T09:52:03Z [INFO] blob sha256:15ad779f15c7efca616eecfc6d848d0c25d3c7aea4dc28e449912b5017814512 of algorithm-rls/lqpt:lqsrv-0.8.6-prod already exists on the destination registry, skip
2018-11-09T09:52:03Z [INFO] blob sha256:8aa6557b859ecc090eed6ae57af0cb3a74ce77cf9829dafa269872cc69f94bfc of algorithm-rls/lqpt:lqsrv-0.8.6-prod already exists on the destination registry, skip
2018-11-09T09:52:03Z [INFO] blob sha256:6e1746509bcd9d1dd8b1382eb44834e425626f4efb842b2f7c129d7c55edef8a of algorithm-rls/lqpt:lqsrv-0.8.6-prod already exists on the destination registry, skip
2018-11-09T09:52:03Z [INFO] blob sha256:500626792d6eb950a4686d5841b41e2f011743d72ee25966213205ad85a76635 of algorithm-rls/lqpt:lqsrv-0.8.6-prod already exists on the destination registry, skip
2018-11-09T09:52:03Z [INFO] blob sha256:500626792d6eb950a4686d5841b41e2f011743d72ee25966213205ad85a76635 of algorithm-rls/lqpt:lqsrv-0.8.6-prod already exists on the destination registry, skip
2018-11-09T09:52:03Z [INFO] manifest of algorithm-rls/lqpt:lqsrv-0.8.6-prod exists on the destination registry, skip manifest pushing
2018-11-09T09:52:03Z [INFO] manifest of algorithm-rls/lqpt:lqsrv-0.8.84 pulled successfully from source registry: sha256:17440df639a4f8a6f81512e451a156554dee32534c641ba06f47128dd7fba89d
2018-11-09T09:52:03Z [INFO] blob sha256:44dec9291879a50e5aa5bd0ab5b2c4e479d804359a7e45580d933da71166627f of algorithm-rls/lqpt:lqsrv-0.8.84 already exists on the destination registry, skip
2018-11-09T09:52:03Z [INFO] blob sha256:16f532fbdc2a4af00095d9a7e72621e401dc2781b264b6351143e7411aa00286 of algorithm-rls/lqpt:lqsrv-0.8.84 already exists on the destination registry, skip
2018-11-09T09:52:04Z [INFO] blob sha256:4301577f4699a2b949a9f5593746658c3a874daa02397214245e7939a67b76ae of algorithm-rls/lqpt:lqsrv-0.8.84 already exists on the destination registry, skip
2018-11-09T09:52:04Z [INFO] blob sha256:047a25444d94b64fbe4c4c8c8c1bd1497910f428056617cc3bac106b17f66c8c of algorithm-rls/lqpt:lqsrv-0.8.84 already exists on the destination registry, skip
2018-11-09T09:52:04Z [INFO] blob sha256:debb691bdbbd071edb18d793f2efaf319c827f5f38c7dd1f070f6f850920fc86 of algorithm-rls/lqpt:lqsrv-0.8.84 already exists on the destination registry, skip
2018-11-09T09:52:04Z [INFO] blob sha256:debb691bdbbd071edb18d793f2efaf319c827f5f38c7dd1f070f6f850920fc86 of algorithm-rls/lqpt:lqsrv-0.8.84 already exists on the destination registry, skip
2018-11-09T09:52:04Z [INFO] manifest of algorithm-rls/lqpt:lqsrv-0.8.84 exists on the destination registry, skip manifest pushing
2018-11-09T09:52:04Z [INFO] manifest of algorithm-rls/lqpt:lqsrv-0.8.9-dev pulled successfully from source registry: sha256:38e8dfe9db5d2aba5f436bb573217679de49f3335a90e3c666a290f57fe620ae
2018-11-09T09:52:04Z [INFO] blob sha256:22e68218750e80b2dd4c23175f2a48598425a1d089f39a4ab23f576afd06cb43 of algorithm-rls/lqpt:lqsrv-0.8.9-dev already exists on the destination registry, skip
2018-11-09T09:52:04Z [INFO] blob sha256:16f532fbdc2a4af00095d9a7e72621e401dc2781b264b6351143e7411aa00286 of algorithm-rls/lqpt:lqsrv-0.8.9-dev already exists on the destination registry, skip
2018-11-09T09:52:04Z [INFO] blob sha256:4301577f4699a2b949a9f5593746658c3a874daa02397214245e7939a67b76ae of algorithm-rls/lqpt:lqsrv-0.8.9-dev already exists on the destination registry, skip
2018-11-09T09:52:04Z [INFO] blob sha256:047a25444d94b64fbe4c4c8c8c1bd1497910f428056617cc3bac106b17f66c8c of algorithm-rls/lqpt:lqsrv-0.8.9-dev already exists on the destination registry, skip
2018-11-09T09:52:04Z [INFO] blob sha256:14602b38395871e064a35054b0d36c91484d2dcf517f5b8147aa55ad567bb6bf of algorithm-rls/lqpt:lqsrv-0.8.9-dev already exists on the destination registry, skip
2018-11-09T09:52:04Z [INFO] blob sha256:6237ba449ce69998a911f5e82b452b70ee72c5fef628a419f47a9acc152528ce of algorithm-rls/lqpt:lqsrv-0.8.9-dev already exists on the destination registry, skip
2018-11-09T09:52:04Z [INFO] manifest of algorithm-rls/lqpt:lqsrv-0.8.9-dev exists on the destination registry, skip manifest pushing
2018-11-09T09:52:05Z [INFO] manifest of algorithm-rls/lqpt:lqsrv-0.9.0-dev pulled successfully from source registry: sha256:f210212245b5f3a075345b078cb109846fa11a8141c9409b7ccf9a5e65418876
2018-11-09T09:52:05Z [INFO] blob sha256:2bb86b0ee46c3bfa64e896db28b4adf76b22c3185a7a01b9f00394f0cf816caa of algorithm-rls/lqpt:lqsrv-0.9.0-dev already exists on the destination registry, skip
2018-11-09T09:52:05Z [INFO] blob sha256:16f532fbdc2a4af00095d9a7e72621e401dc2781b264b6351143e7411aa00286 of algorithm-rls/lqpt:lqsrv-0.9.0-dev already exists on the destination registry, skip
2018-11-09T09:52:05Z [INFO] blob sha256:4301577f4699a2b949a9f5593746658c3a874daa02397214245e7939a67b76ae of algorithm-rls/lqpt:lqsrv-0.9.0-dev already exists on the destination registry, skip
2018-11-09T09:52:05Z [INFO] blob sha256:047a25444d94b64fbe4c4c8c8c1bd1497910f428056617cc3bac106b17f66c8c of algorithm-rls/lqpt:lqsrv-0.9.0-dev already exists on the destination registry, skip
2018-11-09T09:52:05Z [INFO] blob sha256:f9827b66e211c9cad0edfcf2a620f7ed7972f9a6808560bc76c8254dae03dcd7 of algorithm-rls/lqpt:lqsrv-0.9.0-dev already exists on the destination registry, skip
2018-11-09T09:52:05Z [INFO] blob sha256:f9827b66e211c9cad0edfcf2a620f7ed7972f9a6808560bc76c8254dae03dcd7 of algorithm-rls/lqpt:lqsrv-0.9.0-dev already exists on the destination registry, skip
2018-11-09T09:52:05Z [INFO] manifest of algorithm-rls/lqpt:lqsrv-0.9.0-dev exists on the destination registry, skip manifest pushing
2018-11-09T09:52:05Z [INFO] manifest of algorithm-rls/lqpt:lqsrv-0.9.1-dev pulled successfully from source registry: sha256:639fd041b4be51acd999d16cb66c7c5aca551f1dfb6c7792628f3778860fbe67
2018-11-09T09:52:05Z [INFO] blob sha256:28a495ed446159d4b99fe19202577c48f8fd838fa6c3d7ff4ba034db2e2aab89 of algorithm-rls/lqpt:lqsrv-0.9.1-dev already exists on the destination registry, skip
2018-11-09T09:52:05Z [INFO] blob sha256:16f532fbdc2a4af00095d9a7e72621e401dc2781b264b6351143e7411aa00286 of algorithm-rls/lqpt:lqsrv-0.9.1-dev already exists on the destination registry, skip
2018-11-09T09:52:05Z [INFO] blob sha256:4301577f4699a2b949a9f5593746658c3a874daa02397214245e7939a67b76ae of algorithm-rls/lqpt:lqsrv-0.9.1-dev already exists on the destination registry, skip
2018-11-09T09:52:05Z [INFO] blob sha256:047a25444d94b64fbe4c4c8c8c1bd1497910f428056617cc3bac106b17f66c8c of algorithm-rls/lqpt:lqsrv-0.9.1-dev already exists on the destination registry, skip
2018-11-09T09:52:05Z [INFO] blob sha256:82ddf21faf72944c39209c7bbbe35bfb1a9f9036894eb0333e2821baa6ce0c43 of algorithm-rls/lqpt:lqsrv-0.9.1-dev already exists on the destination registry, skip
2018-11-09T09:52:05Z [INFO] blob sha256:82ddf21faf72944c39209c7bbbe35bfb1a9f9036894eb0333e2821baa6ce0c43 of algorithm-rls/lqpt:lqsrv-0.9.1-dev already exists on the destination registry, skip
2018-11-09T09:52:05Z [INFO] manifest of algorithm-rls/lqpt:lqsrv-0.9.1-dev exists on the destination registry, skip manifest pushing
2018-11-09T09:52:05Z [INFO] manifest of algorithm-rls/lqpt:lqsrv-0.9.1-prod pulled successfully from source registry: sha256:ffeb738366ce5a9271fcc5674da6d0cdb0cc46eb898ede6a071019a4e7040d08
2018-11-09T09:52:05Z [INFO] blob sha256:79691a08e7f58f3bf72f417ce86f90503777679cb6778ce06b1ad29a65bf698e of algorithm-rls/lqpt:lqsrv-0.9.1-prod already exists on the destination registry, skip
2018-11-09T09:52:05Z [INFO] blob sha256:16f532fbdc2a4af00095d9a7e72621e401dc2781b264b6351143e7411aa00286 of algorithm-rls/lqpt:lqsrv-0.9.1-prod already exists on the destination registry, skip
2018-11-09T09:52:05Z [INFO] blob sha256:4301577f4699a2b949a9f5593746658c3a874daa02397214245e7939a67b76ae of algorithm-rls/lqpt:lqsrv-0.9.1-prod already exists on the destination registry, skip
2018-11-09T09:52:05Z [INFO] blob sha256:047a25444d94b64fbe4c4c8c8c1bd1497910f428056617cc3bac106b17f66c8c of algorithm-rls/lqpt:lqsrv-0.9.1-prod already exists on the destination registry, skip
2018-11-09T09:52:05Z [INFO] blob sha256:9652a98c1ec08cd4ea8a70eabd305a0d8fe8684008a558cc26b37d6d4c8496fc of algorithm-rls/lqpt:lqsrv-0.9.1-prod already exists on the destination registry, skip
2018-11-09T09:52:05Z [INFO] blob sha256:9652a98c1ec08cd4ea8a70eabd305a0d8fe8684008a558cc26b37d6d4c8496fc of algorithm-rls/lqpt:lqsrv-0.9.1-prod already exists on the destination registry, skip
2018-11-09T09:52:06Z [INFO] manifest of algorithm-rls/lqpt:lqsrv-0.9.1-prod exists on the destination registry, skip manifest pushing
2018-11-09T09:52:06Z [INFO] manifest of algorithm-rls/lqpt:lqsrv-0.9.2-dev pulled successfully from source registry: sha256:b6474a8f44b38867b75154217ba3ad386527198f959a5c3a1f5c431a689bd244
2018-11-09T09:52:06Z [INFO] blob sha256:61a13e7914366b18a913645d38745ac5851349461f3bb665e13187fcc0ee7e3c of algorithm-rls/lqpt:lqsrv-0.9.2-dev already exists on the destination registry, skip
2018-11-09T09:52:06Z [INFO] blob sha256:16f532fbdc2a4af00095d9a7e72621e401dc2781b264b6351143e7411aa00286 of algorithm-rls/lqpt:lqsrv-0.9.2-dev already exists on the destination registry, skip
2018-11-09T09:52:06Z [INFO] blob sha256:4301577f4699a2b949a9f5593746658c3a874daa02397214245e7939a67b76ae of algorithm-rls/lqpt:lqsrv-0.9.2-dev already exists on the destination registry, skip
2018-11-09T09:52:06Z [INFO] blob sha256:047a25444d94b64fbe4c4c8c8c1bd1497910f428056617cc3bac106b17f66c8c of algorithm-rls/lqpt:lqsrv-0.9.2-dev already exists on the destination registry, skip
2018-11-09T09:52:06Z [INFO] blob sha256:f8f1dc55588687ceac1a83dc5e52a2fe508bf85dcfc62761b501543f555ab9b6 of algorithm-rls/lqpt:lqsrv-0.9.2-dev already exists on the destination registry, skip
2018-11-09T09:52:06Z [INFO] blob sha256:1d21dfac3d8439661a1a84c73b057d86bca3b246a81f21a541c3daa9009bd627 of algorithm-rls/lqpt:lqsrv-0.9.2-dev already exists on the destination registry, skip
2018-11-09T09:52:06Z [INFO] manifest of algorithm-rls/lqpt:lqsrv-0.9.2-dev exists on the destination registry, skip manifest pushing
2018-11-09T09:52:06Z [INFO] manifest of algorithm-rls/lqpt:lqsrv-0.9.2-prod pulled successfully from source registry: sha256:065ed637cd67669aa62e04fe2096b6e9bd2ca5fbb5c8fdde9acb0576285dbe57
2018-11-09T09:52:06Z [INFO] blob sha256:569e01d29adb5ecf7d942d8bd5fb60b15db5441ffc1bcce8c1a93409ecae07c8 of algorithm-rls/lqpt:lqsrv-0.9.2-prod already exists on the destination registry, skip
2018-11-09T09:52:06Z [INFO] blob sha256:16f532fbdc2a4af00095d9a7e72621e401dc2781b264b6351143e7411aa00286 of algorithm-rls/lqpt:lqsrv-0.9.2-prod already exists on the destination registry, skip
2018-11-09T09:52:06Z [INFO] blob sha256:4301577f4699a2b949a9f5593746658c3a874daa02397214245e7939a67b76ae of algorithm-rls/lqpt:lqsrv-0.9.2-prod already exists on the destination registry, skip
2018-11-09T09:52:06Z [INFO] blob sha256:047a25444d94b64fbe4c4c8c8c1bd1497910f428056617cc3bac106b17f66c8c of algorithm-rls/lqpt:lqsrv-0.9.2-prod already exists on the destination registry, skip
2018-11-09T09:52:06Z [INFO] blob sha256:60d9b3eae708f35714c5b21d1e9bf22a1db7467928ae7d0212510cacf74644a2 of algorithm-rls/lqpt:lqsrv-0.9.2-prod already exists on the destination registry, skip
2018-11-09T09:52:06Z [INFO] blob sha256:60d9b3eae708f35714c5b21d1e9bf22a1db7467928ae7d0212510cacf74644a2 of algorithm-rls/lqpt:lqsrv-0.9.2-prod already exists on the destination registry, skip
2018-11-09T09:52:06Z [INFO] manifest of algorithm-rls/lqpt:lqsrv-0.9.2-prod exists on the destination registry, skip manifest pushing
2018-11-09T09:52:07Z [INFO] manifest of algorithm-rls/lqpt:lqsrv-0.9.3-prod pulled successfully from source registry: sha256:fb7b0c6cb26662c882d43464cc40de4664dbf93568e3e4efd27e16c9f263ebc1
2018-11-09T09:52:07Z [INFO] blob sha256:a134e9a7c638f948dedc25f0e286709b19439bb96bdd84c0ef395f67b64f428c of algorithm-rls/lqpt:lqsrv-0.9.3-prod already exists on the destination registry, skip
2018-11-09T09:52:07Z [INFO] blob sha256:16f532fbdc2a4af00095d9a7e72621e401dc2781b264b6351143e7411aa00286 of algorithm-rls/lqpt:lqsrv-0.9.3-prod already exists on the destination registry, skip
2018-11-09T09:52:07Z [INFO] blob sha256:4301577f4699a2b949a9f5593746658c3a874daa02397214245e7939a67b76ae of algorithm-rls/lqpt:lqsrv-0.9.3-prod already exists on the destination registry, skip
2018-11-09T09:52:07Z [INFO] blob sha256:047a25444d94b64fbe4c4c8c8c1bd1497910f428056617cc3bac106b17f66c8c of algorithm-rls/lqpt:lqsrv-0.9.3-prod already exists on the destination registry, skip
2018-11-09T09:52:07Z [INFO] blob sha256:0799fe5ac27087ebd6f7268d47f640418c44c3720a17dcd6f9d913ec9df0d638 of algorithm-rls/lqpt:lqsrv-0.9.3-prod already exists on the destination registry, skip
2018-11-09T09:52:07Z [INFO] blob sha256:0799fe5ac27087ebd6f7268d47f640418c44c3720a17dcd6f9d913ec9df0d638 of algorithm-rls/lqpt:lqsrv-0.9.3-prod already exists on the destination registry, skip
2018-11-09T09:52:07Z [INFO] manifest of algorithm-rls/lqpt:lqsrv-0.9.3-prod exists on the destination registry, skip manifest pushing
2018-11-09T09:52:07Z [INFO] manifest of algorithm-rls/lqpt:lqsrv-0.9.4-prod pulled successfully from source registry: sha256:b883066167a4decfea7fb81e3c5bf0853d9b63c81a348e24448e69234a999109
2018-11-09T09:52:07Z [INFO] blob sha256:f44eddf3b68bdf2f4dfdcf53d59dc14f7ae019fd925b85142d3370fc2bd304f1 of algorithm-rls/lqpt:lqsrv-0.9.4-prod already exists on the destination registry, skip
2018-11-09T09:52:07Z [INFO] blob sha256:16f532fbdc2a4af00095d9a7e72621e401dc2781b264b6351143e7411aa00286 of algorithm-rls/lqpt:lqsrv-0.9.4-prod already exists on the destination registry, skip
2018-11-09T09:52:07Z [INFO] blob sha256:4301577f4699a2b949a9f5593746658c3a874daa02397214245e7939a67b76ae of algorithm-rls/lqpt:lqsrv-0.9.4-prod already exists on the destination registry, skip
2018-11-09T09:52:07Z [INFO] blob sha256:047a25444d94b64fbe4c4c8c8c1bd1497910f428056617cc3bac106b17f66c8c of algorithm-rls/lqpt:lqsrv-0.9.4-prod already exists on the destination registry, skip
2018-11-09T09:52:07Z [INFO] blob sha256:154e5127c58b235445692e023b047b1d429f83d356277e219f0595c61a313f91 of algorithm-rls/lqpt:lqsrv-0.9.4-prod already exists on the destination registry, skip
2018-11-09T09:52:07Z [INFO] blob sha256:154e5127c58b235445692e023b047b1d429f83d356277e219f0595c61a313f91 of algorithm-rls/lqpt:lqsrv-0.9.4-prod already exists on the destination registry, skip
2018-11-09T09:52:08Z [INFO] manifest of algorithm-rls/lqpt:lqsrv-0.9.4-prod exists on the destination registry, skip manifest pushing
2018-11-09T09:52:08Z [INFO] manifest of algorithm-rls/lqpt:lqsrv-0.9.5-dev pulled successfully from source registry: sha256:e4572bf0ec371d1e4f52845e4b94f7744c95ded84e2f0fff6d776af85901e90b
2018-11-09T09:52:08Z [INFO] blob sha256:cd09768d1736bd00bfd374981491f1ed1c2c19a91a04379820ebfca9b2bf73cd of algorithm-rls/lqpt:lqsrv-0.9.5-dev already exists on the destination registry, skip
2018-11-09T09:52:08Z [INFO] blob sha256:04651435ae6137b12ecdb600b699400834c0c78201d7ffa24ac0593638c4346d of algorithm-rls/lqpt:lqsrv-0.9.5-dev already exists on the destination registry, skip
2018-11-09T09:52:08Z [INFO] blob sha256:ccae121f92fdaaea7e4da27a848efa78adf3cc940b1a08bbcf3583de5f8ae65b of algorithm-rls/lqpt:lqsrv-0.9.5-dev already exists on the destination registry, skip
2018-11-09T09:52:08Z [INFO] blob sha256:7bb876499e214699ab8c21364ead2b3a227edd9a8698c6936b4350e778b73473 of algorithm-rls/lqpt:lqsrv-0.9.5-dev already exists on the destination registry, skip
2018-11-09T09:52:08Z [INFO] blob sha256:444d1ce6037a420ad7ffea67e8072c699c055910e5ef60720959e37a1ec4047a of algorithm-rls/lqpt:lqsrv-0.9.5-dev already exists on the destination registry, skip
2018-11-09T09:52:08Z [INFO] blob sha256:f02a7b59a9fd6db04b3958bedca267f7e2510897a5ed8e8741ee7dd9701dd479 of algorithm-rls/lqpt:lqsrv-0.9.5-dev already exists on the destination registry, skip
2018-11-09T09:52:08Z [INFO] blob sha256:1ce18ba1d5769b3619946005b8e008519f39cce67988532d920502d9b58fbd0e of algorithm-rls/lqpt:lqsrv-0.9.5-dev already exists on the destination registry, skip
2018-11-09T09:52:08Z [INFO] blob sha256:d007c42ebaa7a477531b8423af514ea13bf07114503874b3166a9affe84df889 of algorithm-rls/lqpt:lqsrv-0.9.5-dev already exists on the destination registry, skip
2018-11-09T09:52:08Z [INFO] blob sha256:bf73c104e7b503f33f07a86dcc0cba4064546929554ec1e3e50bc059c9d279a4 of algorithm-rls/lqpt:lqsrv-0.9.5-dev already exists on the destination registry, skip
2018-11-09T09:52:08Z [INFO] blob sha256:aa0dbceb79879ae9c1658767c1d20c4784312b1f898f6b5fb77bfb9c3b6ad32e of algorithm-rls/lqpt:lqsrv-0.9.5-dev already exists on the destination registry, skip
2018-11-09T09:52:08Z [INFO] blob sha256:3b2522307adf71e17e8feb2cd1da49aee06b9de61590e8bf4f1ae0dc7bb259d8 of algorithm-rls/lqpt:lqsrv-0.9.5-dev already exists on the destination registry, skip
2018-11-09T09:52:08Z [INFO] manifest of algorithm-rls/lqpt:lqsrv-0.9.5-dev exists on the destination registry, skip manifest pushing
2018-11-09T09:52:08Z [INFO] manifest of algorithm-rls/lqpt:lqsrv-0.9.6-dev pulled successfully from source registry: sha256:7747c6b3d2b3346c0b29ae1b0070c0a1b0ab83ac526d245fd222a07f0de01861
2018-11-09T09:52:08Z [INFO] blob sha256:167f4c1f50652bb1de0d74f4d3e38e844944bdfcd1d779ff832983b3309992b6 of algorithm-rls/lqpt:lqsrv-0.9.6-dev already exists on the destination registry, skip
2018-11-09T09:52:08Z [INFO] blob sha256:04651435ae6137b12ecdb600b699400834c0c78201d7ffa24ac0593638c4346d of algorithm-rls/lqpt:lqsrv-0.9.6-dev already exists on the destination registry, skip
2018-11-09T09:52:08Z [INFO] blob sha256:ccae121f92fdaaea7e4da27a848efa78adf3cc940b1a08bbcf3583de5f8ae65b of algorithm-rls/lqpt:lqsrv-0.9.6-dev already exists on the destination registry, skip
2018-11-09T09:52:08Z [INFO] blob sha256:7bb876499e214699ab8c21364ead2b3a227edd9a8698c6936b4350e778b73473 of algorithm-rls/lqpt:lqsrv-0.9.6-dev already exists on the destination registry, skip
2018-11-09T09:52:09Z [INFO] blob sha256:444d1ce6037a420ad7ffea67e8072c699c055910e5ef60720959e37a1ec4047a of algorithm-rls/lqpt:lqsrv-0.9.6-dev already exists on the destination registry, skip
2018-11-09T09:52:09Z [INFO] blob sha256:f02a7b59a9fd6db04b3958bedca267f7e2510897a5ed8e8741ee7dd9701dd479 of algorithm-rls/lqpt:lqsrv-0.9.6-dev already exists on the destination registry, skip
2018-11-09T09:52:09Z [INFO] blob sha256:801ab075786944e7066153e794a851778c99b949887449488f8e179bbb964aec of algorithm-rls/lqpt:lqsrv-0.9.6-dev already exists on the destination registry, skip
2018-11-09T09:52:09Z [INFO] blob sha256:9766a82f099271e4b6eff0a151d2c81fc651d487e744366e016c8b31ab3d2d49 of algorithm-rls/lqpt:lqsrv-0.9.6-dev already exists on the destination registry, skip
2018-11-09T09:52:09Z [INFO] blob sha256:37a4892691d9d8f67394c1fad53e8aa70884c33e57135530807287e20850b213 of algorithm-rls/lqpt:lqsrv-0.9.6-dev already exists on the destination registry, skip
2018-11-09T09:52:09Z [INFO] blob sha256:a2a79ccfb388699d5bf24dde0f6eccad821bad626c19079a4921121527652afa of algorithm-rls/lqpt:lqsrv-0.9.6-dev already exists on the destination registry, skip
2018-11-09T09:52:09Z [INFO] blob sha256:16df69838d2437daaeedb0c8591cd7d70cd42c33c3ea690bdc9a69655fe9a59f of algorithm-rls/lqpt:lqsrv-0.9.6-dev already exists on the destination registry, skip
2018-11-09T09:52:09Z [INFO] manifest of algorithm-rls/lqpt:lqsrv-0.9.6-dev exists on the destination registry, skip manifest pushing
2018-11-09T09:52:09Z [INFO] manifest of algorithm-rls/lqpt:lqsrv-0.9.7-dev pulled successfully from source registry: sha256:6f55850b20fe7bd2219ca1e1640159eeedb6f9450b1a73793ac15f429534ded6
2018-11-09T09:52:09Z [INFO] blob sha256:501312c07156ca902af915b69160e979158603ccea849a2daa0a581834aea51e of algorithm-rls/lqpt:lqsrv-0.9.7-dev already exists on the destination registry, skip
2018-11-09T09:52:09Z [INFO] blob sha256:04651435ae6137b12ecdb600b699400834c0c78201d7ffa24ac0593638c4346d of algorithm-rls/lqpt:lqsrv-0.9.7-dev already exists on the destination registry, skip
2018-11-09T09:52:09Z [INFO] blob sha256:ccae121f92fdaaea7e4da27a848efa78adf3cc940b1a08bbcf3583de5f8ae65b of algorithm-rls/lqpt:lqsrv-0.9.7-dev already exists on the destination registry, skip
2018-11-09T09:52:09Z [INFO] blob sha256:7bb876499e214699ab8c21364ead2b3a227edd9a8698c6936b4350e778b73473 of algorithm-rls/lqpt:lqsrv-0.9.7-dev already exists on the destination registry, skip
2018-11-09T09:52:09Z [INFO] blob sha256:444d1ce6037a420ad7ffea67e8072c699c055910e5ef60720959e37a1ec4047a of algorithm-rls/lqpt:lqsrv-0.9.7-dev already exists on the destination registry, skip
2018-11-09T09:52:09Z [INFO] blob sha256:f02a7b59a9fd6db04b3958bedca267f7e2510897a5ed8e8741ee7dd9701dd479 of algorithm-rls/lqpt:lqsrv-0.9.7-dev already exists on the destination registry, skip
2018-11-09T09:52:09Z [INFO] blob sha256:9b6b59468f4be144d0ad2c2a8937deaec938a0dec210f27ee9f87849cafb177d of algorithm-rls/lqpt:lqsrv-0.9.7-dev already exists on the destination registry, skip
2018-11-09T09:52:09Z [INFO] blob sha256:8e27554c4063b04f5d88de9135678384b929f6376a50a143adf6a105d7e92dd5 of algorithm-rls/lqpt:lqsrv-0.9.7-dev already exists on the destination registry, skip
2018-11-09T09:52:09Z [INFO] blob sha256:f84d3b4cdf22d8f0a0d9bc04fed14713b3db88f093a1a920c44e5b39cc737925 of algorithm-rls/lqpt:lqsrv-0.9.7-dev already exists on the destination registry, skip
2018-11-09T09:52:09Z [INFO] blob sha256:484ee5cc88fc9a19152e70d0ae8440882e96f19c84178aeeb3cf6450dadc1125 of algorithm-rls/lqpt:lqsrv-0.9.7-dev already exists on the destination registry, skip
2018-11-09T09:52:09Z [INFO] blob sha256:ab1077e8f568e0e1efd46bad5aa1b38b9d05698d4cb5caef638eb9339ed650a1 of algorithm-rls/lqpt:lqsrv-0.9.7-dev already exists on the destination registry, skip
2018-11-09T09:52:09Z [INFO] manifest of algorithm-rls/lqpt:lqsrv-0.9.7-dev exists on the destination registry, skip manifest pushing
2018-11-09T09:52:09Z [INFO] manifest of algorithm-rls/lqpt:lqsrv-0.9.7-prod pulled successfully from source registry: sha256:389924600edfbdfac011c0c9a22ab4b5beaac927f0d2722d489e52dc8bd0bbe2
2018-11-09T09:52:09Z [INFO] blob sha256:f65a0e594eae196c405ff86c3a5271e31663ce4ad17a438b15332ddbbf93ac20 of algorithm-rls/lqpt:lqsrv-0.9.7-prod already exists on the destination registry, skip
2018-11-09T09:52:10Z [INFO] blob sha256:04651435ae6137b12ecdb600b699400834c0c78201d7ffa24ac0593638c4346d of algorithm-rls/lqpt:lqsrv-0.9.7-prod already exists on the destination registry, skip
2018-11-09T09:52:10Z [INFO] blob sha256:ccae121f92fdaaea7e4da27a848efa78adf3cc940b1a08bbcf3583de5f8ae65b of algorithm-rls/lqpt:lqsrv-0.9.7-prod already exists on the destination registry, skip
2018-11-09T09:52:10Z [INFO] blob sha256:7bb876499e214699ab8c21364ead2b3a227edd9a8698c6936b4350e778b73473 of algorithm-rls/lqpt:lqsrv-0.9.7-prod already exists on the destination registry, skip
2018-11-09T09:52:10Z [INFO] blob sha256:444d1ce6037a420ad7ffea67e8072c699c055910e5ef60720959e37a1ec4047a of algorithm-rls/lqpt:lqsrv-0.9.7-prod already exists on the destination registry, skip
2018-11-09T09:52:10Z [INFO] blob sha256:f02a7b59a9fd6db04b3958bedca267f7e2510897a5ed8e8741ee7dd9701dd479 of algorithm-rls/lqpt:lqsrv-0.9.7-prod already exists on the destination registry, skip
2018-11-09T09:52:10Z [INFO] blob sha256:b196db0dbd6b582e2e97e2c43e773f1ce6ebdcf706bcc8b5c206b819bcfe7249 of algorithm-rls/lqpt:lqsrv-0.9.7-prod already exists on the destination registry, skip
2018-11-09T09:52:10Z [INFO] blob sha256:8fb8fd5cd596bea4a07c1c6c10315e12bbae51b967e6e1716ede5cfa96bb5246 of algorithm-rls/lqpt:lqsrv-0.9.7-prod already exists on the destination registry, skip
2018-11-09T09:52:10Z [INFO] blob sha256:fef640d116f0300ce75f3d2209e415c44c3219023ce8885f5a75a331bcc413c3 of algorithm-rls/lqpt:lqsrv-0.9.7-prod already exists on the destination registry, skip
2018-11-09T09:52:10Z [INFO] blob sha256:ba613d361a7505886e4568ae6dd8e73a7d444d9d413e0906258f35e755ba53a0 of algorithm-rls/lqpt:lqsrv-0.9.7-prod already exists on the destination registry, skip
2018-11-09T09:52:10Z [INFO] blob sha256:745a542d0955d76e0237c3fc633549bb00a0513e4e280a9206f7bf9b9e9883fe of algorithm-rls/lqpt:lqsrv-0.9.7-prod already exists on the destination registry, skip
2018-11-09T09:52:10Z [INFO] manifest of algorithm-rls/lqpt:lqsrv-0.9.7-prod exists on the destination registry, skip manifest pushing
2018-11-09T09:52:10Z [INFO] manifest of algorithm-rls/lqpt:lqsrv-1.0.1-dev pulled successfully from source registry: sha256:c8aba8f9df65ba53fd2be42117b9e2d9fb1796cf8d660a4b6b55b4abe3062723
2018-11-09T09:52:10Z [INFO] blob sha256:ca351b452b200bf767e9dc8161c8fe847c907dbd8cec82f48d6a1e6770fd0396 of algorithm-rls/lqpt:lqsrv-1.0.1-dev already exists on the destination registry, skip
2018-11-09T09:52:10Z [INFO] blob sha256:04651435ae6137b12ecdb600b699400834c0c78201d7ffa24ac0593638c4346d of algorithm-rls/lqpt:lqsrv-1.0.1-dev already exists on the destination registry, skip
2018-11-09T09:52:10Z [INFO] blob sha256:ccae121f92fdaaea7e4da27a848efa78adf3cc940b1a08bbcf3583de5f8ae65b of algorithm-rls/lqpt:lqsrv-1.0.1-dev already exists on the destination registry, skip
2018-11-09T09:52:10Z [INFO] blob sha256:7bb876499e214699ab8c21364ead2b3a227edd9a8698c6936b4350e778b73473 of algorithm-rls/lqpt:lqsrv-1.0.1-dev already exists on the destination registry, skip
2018-11-09T09:52:10Z [INFO] blob sha256:444d1ce6037a420ad7ffea67e8072c699c055910e5ef60720959e37a1ec4047a of algorithm-rls/lqpt:lqsrv-1.0.1-dev already exists on the destination registry, skip
2018-11-09T09:52:10Z [INFO] blob sha256:f02a7b59a9fd6db04b3958bedca267f7e2510897a5ed8e8741ee7dd9701dd479 of algorithm-rls/lqpt:lqsrv-1.0.1-dev already exists on the destination registry, skip
2018-11-09T09:52:10Z [INFO] blob sha256:cb3660ad40b4275bd20ec14df185260b00c8cc1b1fa8d352b7f79e8366ad62dc of algorithm-rls/lqpt:lqsrv-1.0.1-dev already exists on the destination registry, skip
2018-11-09T09:52:10Z [INFO] blob sha256:ecd0f69e178c61b9a6c36ca68e21b367217c8714ecd24ab7b0c391a7565bdbf8 of algorithm-rls/lqpt:lqsrv-1.0.1-dev already exists on the destination registry, skip
2018-11-09T09:52:10Z [INFO] blob sha256:676f13e3151b29fbf4bf2df038b326b860f06ccdf366d4d9b2fa9d9a1c6966b6 of algorithm-rls/lqpt:lqsrv-1.0.1-dev already exists on the destination registry, skip
2018-11-09T09:52:10Z [INFO] blob sha256:34689f696ed75e3b2ff0421805066d56a88f292abd7195bac39ce422c5051be1 of algorithm-rls/lqpt:lqsrv-1.0.1-dev already exists on the destination registry, skip
2018-11-09T09:52:10Z [INFO] blob sha256:7111367ffa1cab0271bea2a518b4e2565c62af06cd0ffd96ac905d82f9dfef6f of algorithm-rls/lqpt:lqsrv-1.0.1-dev already exists on the destination registry, skip
2018-11-09T09:52:10Z [INFO] blob sha256:734171647f9c8de2b62ddd9f209290a573ae76d3e99e64240cc98dd199023b41 of algorithm-rls/lqpt:lqsrv-1.0.1-dev already exists on the destination registry, skip
2018-11-09T09:52:10Z [INFO] manifest of algorithm-rls/lqpt:lqsrv-1.0.1-dev exists on the destination registry, skip manifest pushing
2018-11-09T09:52:11Z [INFO] manifest of algorithm-rls/lqpt:lqsrv-1.0.2-dev pulled successfully from source registry: sha256:3cbed8b337fe23c148e8fd885303974e58c5bb1f7149d9caddbbb60cb905c8dd
2018-11-09T09:52:11Z [INFO] blob sha256:c04e10f0f3d3f2ae7d4e51e43d3193e65b4f927911508d7bc6ac71da081c737e of algorithm-rls/lqpt:lqsrv-1.0.2-dev already exists on the destination registry, skip
2018-11-09T09:52:11Z [INFO] blob sha256:04651435ae6137b12ecdb600b699400834c0c78201d7ffa24ac0593638c4346d of algorithm-rls/lqpt:lqsrv-1.0.2-dev already exists on the destination registry, skip
2018-11-09T09:52:11Z [INFO] blob sha256:ccae121f92fdaaea7e4da27a848efa78adf3cc940b1a08bbcf3583de5f8ae65b of algorithm-rls/lqpt:lqsrv-1.0.2-dev already exists on the destination registry, skip
2018-11-09T09:52:11Z [INFO] blob sha256:7bb876499e214699ab8c21364ead2b3a227edd9a8698c6936b4350e778b73473 of algorithm-rls/lqpt:lqsrv-1.0.2-dev already exists on the destination registry, skip
2018-11-09T09:52:11Z [INFO] blob sha256:444d1ce6037a420ad7ffea67e8072c699c055910e5ef60720959e37a1ec4047a of algorithm-rls/lqpt:lqsrv-1.0.2-dev already exists on the destination registry, skip
2018-11-09T09:52:11Z [INFO] blob sha256:f02a7b59a9fd6db04b3958bedca267f7e2510897a5ed8e8741ee7dd9701dd479 of algorithm-rls/lqpt:lqsrv-1.0.2-dev already exists on the destination registry, skip
2018-11-09T09:52:11Z [INFO] blob sha256:72616a47aa23b6241794cdcae53752ca9083de2bae0e9fea468b2de436fa5e96 of algorithm-rls/lqpt:lqsrv-1.0.2-dev already exists on the destination registry, skip
2018-11-09T09:52:11Z [INFO] blob sha256:1f5a76af92027cfdf29c57b180db1ff7615e6e0757d79120d7cfc51e97cf02de of algorithm-rls/lqpt:lqsrv-1.0.2-dev already exists on the destination registry, skip
2018-11-09T09:52:11Z [INFO] blob sha256:cd157821537a6d170b6bbd40406bfa15d2d74db12163ff0a29ac053108fd8111 of algorithm-rls/lqpt:lqsrv-1.0.2-dev already exists on the destination registry, skip
2018-11-09T09:52:11Z [INFO] blob sha256:badfef6cff0c8a2eb68c7794abe62b1287b15a697eb1bef581e4443790761c8e of algorithm-rls/lqpt:lqsrv-1.0.2-dev already exists on the destination registry, skip
2018-11-09T09:52:11Z [INFO] blob sha256:40494798cd9f3c666478f8954bff71981c93c9e06631805f4e4c9b57b3be53f5 of algorithm-rls/lqpt:lqsrv-1.0.2-dev already exists on the destination registry, skip
2018-11-09T09:52:11Z [INFO] blob sha256:9bf37452c9a4bd878015ca8d5abaf9e5e33cce7ce240501cd2b8514f6cc13bf9 of algorithm-rls/lqpt:lqsrv-1.0.2-dev already exists on the destination registry, skip
2018-11-09T09:52:11Z [INFO] manifest of algorithm-rls/lqpt:lqsrv-1.0.2-dev exists on the destination registry, skip manifest pushing
2018-11-09T09:52:11Z [INFO] manifest of algorithm-rls/lqpt:lqsrv-1.0.2-prod pulled successfully from source registry: sha256:cdba892592e3f4ba3be36a1e2cae51ec6202748424210fbe825abf3b89b1465c
2018-11-09T09:52:11Z [INFO] blob sha256:1e3434691082be862c398f1a650131e7b94c017b8fc9c34e531c691ccf47ad6e of algorithm-rls/lqpt:lqsrv-1.0.2-prod already exists on the destination registry, skip
2018-11-09T09:52:11Z [INFO] blob sha256:04651435ae6137b12ecdb600b699400834c0c78201d7ffa24ac0593638c4346d of algorithm-rls/lqpt:lqsrv-1.0.2-prod already exists on the destination registry, skip
2018-11-09T09:52:11Z [INFO] blob sha256:ccae121f92fdaaea7e4da27a848efa78adf3cc940b1a08bbcf3583de5f8ae65b of algorithm-rls/lqpt:lqsrv-1.0.2-prod already exists on the destination registry, skip
2018-11-09T09:52:11Z [INFO] blob sha256:7bb876499e214699ab8c21364ead2b3a227edd9a8698c6936b4350e778b73473 of algorithm-rls/lqpt:lqsrv-1.0.2-prod already exists on the destination registry, skip
2018-11-09T09:52:11Z [INFO] blob sha256:444d1ce6037a420ad7ffea67e8072c699c055910e5ef60720959e37a1ec4047a of algorithm-rls/lqpt:lqsrv-1.0.2-prod already exists on the destination registry, skip
2018-11-09T09:52:11Z [INFO] blob sha256:f02a7b59a9fd6db04b3958bedca267f7e2510897a5ed8e8741ee7dd9701dd479 of algorithm-rls/lqpt:lqsrv-1.0.2-prod already exists on the destination registry, skip
2018-11-09T09:52:11Z [INFO] blob sha256:a2bcd02710efdb3ca8f664deefc9213eef102566eb448938cfbd15e2da370cfd of algorithm-rls/lqpt:lqsrv-1.0.2-prod already exists on the destination registry, skip
2018-11-09T09:52:11Z [INFO] blob sha256:4e67b8596af2619fcc41bc45cb3ce9e832ccd0d534c0ad2e7de35e468e159593 of algorithm-rls/lqpt:lqsrv-1.0.2-prod already exists on the destination registry, skip
2018-11-09T09:52:11Z [INFO] blob sha256:c68488e5201549ee2762c413b7a6b76b7d715c822140ea8c94b442f6dda4df44 of algorithm-rls/lqpt:lqsrv-1.0.2-prod already exists on the destination registry, skip
2018-11-09T09:52:11Z [INFO] blob sha256:d47cc3686e5e5ffad06071598cc3d48ec017cd5e1fbb4e3678f202d010ef171a of algorithm-rls/lqpt:lqsrv-1.0.2-prod already exists on the destination registry, skip
2018-11-09T09:52:11Z [INFO] blob sha256:0a8e06f10dbde9ed2d49466b2d2d26062b8b8b0a8e548ccc82db62e54a02d503 of algorithm-rls/lqpt:lqsrv-1.0.2-prod already exists on the destination registry, skip
2018-11-09T09:52:11Z [INFO] blob sha256:f6ee6c01dc6795462b4304ccff7f84f2f22e77cc48d06f0928550f5267ee6f82 of algorithm-rls/lqpt:lqsrv-1.0.2-prod already exists on the destination registry, skip
2018-11-09T09:52:12Z [INFO] manifest of algorithm-rls/lqpt:lqsrv-1.0.2-prod exists on the destination registry, skip manifest pushing
2018-11-09T09:52:12Z [INFO] manifest of algorithm-rls/lqpt:lqsrv-1.0.3-dev pulled successfully from source registry: sha256:f5f9d19a00b19dde772756f4beebda7f08b74c87956729ec532cc5f70f6610ae
2018-11-09T09:52:12Z [INFO] blob sha256:330cbf8dc4f727b373de15fa975044915581065e5a4ae8fd400d6f168e6d1035 of algorithm-rls/lqpt:lqsrv-1.0.3-dev already exists on the destination registry, skip
2018-11-09T09:52:12Z [INFO] blob sha256:04651435ae6137b12ecdb600b699400834c0c78201d7ffa24ac0593638c4346d of algorithm-rls/lqpt:lqsrv-1.0.3-dev already exists on the destination registry, skip
2018-11-09T09:52:12Z [INFO] blob sha256:ccae121f92fdaaea7e4da27a848efa78adf3cc940b1a08bbcf3583de5f8ae65b of algorithm-rls/lqpt:lqsrv-1.0.3-dev already exists on the destination registry, skip
2018-11-09T09:52:12Z [INFO] blob sha256:7bb876499e214699ab8c21364ead2b3a227edd9a8698c6936b4350e778b73473 of algorithm-rls/lqpt:lqsrv-1.0.3-dev already exists on the destination registry, skip
2018-11-09T09:52:12Z [INFO] blob sha256:444d1ce6037a420ad7ffea67e8072c699c055910e5ef60720959e37a1ec4047a of algorithm-rls/lqpt:lqsrv-1.0.3-dev already exists on the destination registry, skip
2018-11-09T09:52:12Z [INFO] blob sha256:f02a7b59a9fd6db04b3958bedca267f7e2510897a5ed8e8741ee7dd9701dd479 of algorithm-rls/lqpt:lqsrv-1.0.3-dev already exists on the destination registry, skip
2018-11-09T09:52:12Z [INFO] blob sha256:42bc6ca85a9e2153d864d2792f1a6259c3db29bee219ce1f3eb4f037a89d34d0 of algorithm-rls/lqpt:lqsrv-1.0.3-dev already exists on the destination registry, skip
2018-11-09T09:52:12Z [INFO] blob sha256:0058175907d3ba42c01c3b5a1b14b175c4279e991ec64d04e45d7e5ae7f72632 of algorithm-rls/lqpt:lqsrv-1.0.3-dev already exists on the destination registry, skip
2018-11-09T09:52:12Z [INFO] blob sha256:295fdf0e3ba3f2278f6dfcc0c1794ebd6f10c8b2966a44a0e5aaaf10a71cc40e of algorithm-rls/lqpt:lqsrv-1.0.3-dev already exists on the destination registry, skip
2018-11-09T09:52:12Z [INFO] blob sha256:9435e285a27f9a5d0ad15d23bb5e6f1525ac08ba4631ff53936bec50350dd5f7 of algorithm-rls/lqpt:lqsrv-1.0.3-dev already exists on the destination registry, skip
2018-11-09T09:52:12Z [INFO] blob sha256:4baded599ea3a3a18b8ccf2ba5b6e464ca3b9127e60d07e7fa9ec0e34718aa3f of algorithm-rls/lqpt:lqsrv-1.0.3-dev already exists on the destination registry, skip
2018-11-09T09:52:12Z [INFO] blob sha256:7c399d4208400507546d9a4ffb7ac92d46a0968cbe9a1985ff56c97cf0df1da2 of algorithm-rls/lqpt:lqsrv-1.0.3-dev already exists on the destination registry, skip
2018-11-09T09:52:12Z [INFO] manifest of algorithm-rls/lqpt:lqsrv-1.0.3-dev exists on the destination registry, skip manifest pushing
2018-11-09T09:52:12Z [INFO] manifest of algorithm-rls/lqpt:lqsrv-1.0.3-prod pulled successfully from source registry: sha256:7ca7c3713669b836f2b492825e0f61eab3156d6f958d262890d9e4298082e757
2018-11-09T09:52:12Z [INFO] blob sha256:802355cd79626e0d142813a749ab0bdf815fb2c36de9cbb2fd225dc31ca80660 of algorithm-rls/lqpt:lqsrv-1.0.3-prod already exists on the destination registry, skip
2018-11-09T09:52:12Z [INFO] blob sha256:04651435ae6137b12ecdb600b699400834c0c78201d7ffa24ac0593638c4346d of algorithm-rls/lqpt:lqsrv-1.0.3-prod already exists on the destination registry, skip
2018-11-09T09:52:12Z [INFO] blob sha256:ccae121f92fdaaea7e4da27a848efa78adf3cc940b1a08bbcf3583de5f8ae65b of algorithm-rls/lqpt:lqsrv-1.0.3-prod already exists on the destination registry, skip
2018-11-09T09:52:12Z [INFO] blob sha256:7bb876499e214699ab8c21364ead2b3a227edd9a8698c6936b4350e778b73473 of algorithm-rls/lqpt:lqsrv-1.0.3-prod already exists on the destination registry, skip
2018-11-09T09:52:12Z [INFO] blob sha256:444d1ce6037a420ad7ffea67e8072c699c055910e5ef60720959e37a1ec4047a of algorithm-rls/lqpt:lqsrv-1.0.3-prod already exists on the destination registry, skip
2018-11-09T09:52:12Z [INFO] blob sha256:f02a7b59a9fd6db04b3958bedca267f7e2510897a5ed8e8741ee7dd9701dd479 of algorithm-rls/lqpt:lqsrv-1.0.3-prod already exists on the destination registry, skip
2018-11-09T09:52:12Z [INFO] blob sha256:eb9b12889678d34875f49104f23e1406258403c1960b32ef4220d91c729d1849 of algorithm-rls/lqpt:lqsrv-1.0.3-prod already exists on the destination registry, skip
2018-11-09T09:52:13Z [INFO] blob sha256:d4d422a05c67347e1220a672f9978e2ec265b8e9aba5dba1e1697595de4a0986 of algorithm-rls/lqpt:lqsrv-1.0.3-prod already exists on the destination registry, skip
2018-11-09T09:52:13Z [INFO] blob sha256:f58e5f44296962c73c7286e742185aa993fdf96c8891a7f879a97fb3ff52420c of algorithm-rls/lqpt:lqsrv-1.0.3-prod already exists on the destination registry, skip
2018-11-09T09:52:13Z [INFO] blob sha256:a6f5e25182a24d0df3d3fd5e3789985aeff0fc9f6fb0d25509a29108a6ad1864 of algorithm-rls/lqpt:lqsrv-1.0.3-prod already exists on the destination registry, skip
2018-11-09T09:52:13Z [INFO] blob sha256:5503c68b1395f8367a95e6f7288ea1f9b27e2d320bd05fe37bbc468503f967a1 of algorithm-rls/lqpt:lqsrv-1.0.3-prod already exists on the destination registry, skip
2018-11-09T09:52:13Z [INFO] blob sha256:faaca53920fa9abde828f74095f63618f9ca557f214181793054ae0771d1ce35 of algorithm-rls/lqpt:lqsrv-1.0.3-prod already exists on the destination registry, skip
2018-11-09T09:52:13Z [INFO] manifest of algorithm-rls/lqpt:lqsrv-1.0.3-prod exists on the destination registry, skip manifest pushing
2018-11-09T09:52:13Z [INFO] manifest of algorithm-rls/lqpt:lqsrv-1.0.4-dev pulled successfully from source registry: sha256:dcef1d52854311818389eaa19e4908c4721ff086c6b102ec04b27f66680b790a
2018-11-09T09:52:13Z [INFO] blob sha256:567b571afe59ef047910824864ee3e527c2616d646360f2633b3014d5595b6cb of algorithm-rls/lqpt:lqsrv-1.0.4-dev already exists on the destination registry, skip
2018-11-09T09:52:13Z [INFO] blob sha256:04651435ae6137b12ecdb600b699400834c0c78201d7ffa24ac0593638c4346d of algorithm-rls/lqpt:lqsrv-1.0.4-dev already exists on the destination registry, skip
2018-11-09T09:52:13Z [INFO] blob sha256:ccae121f92fdaaea7e4da27a848efa78adf3cc940b1a08bbcf3583de5f8ae65b of algorithm-rls/lqpt:lqsrv-1.0.4-dev already exists on the destination registry, skip
2018-11-09T09:52:13Z [INFO] blob sha256:7bb876499e214699ab8c21364ead2b3a227edd9a8698c6936b4350e778b73473 of algorithm-rls/lqpt:lqsrv-1.0.4-dev already exists on the destination registry, skip
2018-11-09T09:52:13Z [INFO] blob sha256:444d1ce6037a420ad7ffea67e8072c699c055910e5ef60720959e37a1ec4047a of algorithm-rls/lqpt:lqsrv-1.0.4-dev already exists on the destination registry, skip
2018-11-09T09:52:13Z [INFO] blob sha256:f02a7b59a9fd6db04b3958bedca267f7e2510897a5ed8e8741ee7dd9701dd479 of algorithm-rls/lqpt:lqsrv-1.0.4-dev already exists on the destination registry, skip
2018-11-09T09:52:13Z [INFO] blob sha256:5ab1fb9535199fc7b5cc61a188795e7297ed56990f732da7fa2cc5355b06b7c3 of algorithm-rls/lqpt:lqsrv-1.0.4-dev already exists on the destination registry, skip
2018-11-09T09:52:13Z [INFO] blob sha256:d9ba1f8ca34db49e8ad3a7581fe026c9410b5fc39467c875763c762e914c3c29 of algorithm-rls/lqpt:lqsrv-1.0.4-dev already exists on the destination registry, skip
2018-11-09T09:52:13Z [INFO] blob sha256:c3918f5813d9544b3fd14335de6e1ecb66af9bd4b0df70f877d5387bd2a5bd26 of algorithm-rls/lqpt:lqsrv-1.0.4-dev already exists on the destination registry, skip
2018-11-09T09:52:13Z [INFO] blob sha256:2e5e3eb73eec78716f6c4e5e68f232123bb7db856a55a116a50997c133935f99 of algorithm-rls/lqpt:lqsrv-1.0.4-dev already exists on the destination registry, skip
2018-11-09T09:52:13Z [INFO] blob sha256:0303ca7b28fa5dbad8a595c87432e275c2eb26312effdd0c5ada376beb3988e2 of algorithm-rls/lqpt:lqsrv-1.0.4-dev already exists on the destination registry, skip
2018-11-09T09:52:13Z [INFO] blob sha256:7b0be3f66fce013bc0ff1cfb88e502a8498e1fb7a12fd30c446d2388bef1fb53 of algorithm-rls/lqpt:lqsrv-1.0.4-dev already exists on the destination registry, skip
2018-11-09T09:52:13Z [INFO] manifest of algorithm-rls/lqpt:lqsrv-1.0.4-dev exists on the destination registry, skip manifest pushing
2018-11-09T09:52:13Z [INFO] manifest of algorithm-rls/lqpt:lqsrv-1.0.4-prod pulled successfully from source registry: sha256:d7fea374c46f15f017cd0b96263ac5b87481db029e10f29d67123e53ebf9f6aa
2018-11-09T09:52:13Z [INFO] blob sha256:bde6bb0c22fc7df33d389c3d8c821294063c147380ce4edfeabdce2729fb8250 of algorithm-rls/lqpt:lqsrv-1.0.4-prod already exists on the destination registry, skip
2018-11-09T09:52:13Z [INFO] blob sha256:04651435ae6137b12ecdb600b699400834c0c78201d7ffa24ac0593638c4346d of algorithm-rls/lqpt:lqsrv-1.0.4-prod already exists on the destination registry, skip
2018-11-09T09:52:13Z [INFO] blob sha256:ccae121f92fdaaea7e4da27a848efa78adf3cc940b1a08bbcf3583de5f8ae65b of algorithm-rls/lqpt:lqsrv-1.0.4-prod already exists on the destination registry, skip
2018-11-09T09:52:13Z [INFO] blob sha256:7bb876499e214699ab8c21364ead2b3a227edd9a8698c6936b4350e778b73473 of algorithm-rls/lqpt:lqsrv-1.0.4-prod already exists on the destination registry, skip
2018-11-09T09:52:14Z [INFO] blob sha256:444d1ce6037a420ad7ffea67e8072c699c055910e5ef60720959e37a1ec4047a of algorithm-rls/lqpt:lqsrv-1.0.4-prod already exists on the destination registry, skip
2018-11-09T09:52:14Z [INFO] blob sha256:f02a7b59a9fd6db04b3958bedca267f7e2510897a5ed8e8741ee7dd9701dd479 of algorithm-rls/lqpt:lqsrv-1.0.4-prod already exists on the destination registry, skip
2018-11-09T09:52:14Z [INFO] blob sha256:292802863351ea2ad8299f6a61bd6b32d699e61cd9231e60dbedd6c97750b208 of algorithm-rls/lqpt:lqsrv-1.0.4-prod already exists on the destination registry, skip
2018-11-09T09:52:14Z [INFO] blob sha256:cfc2788f935c7d904b3148c0bfc86691b0ceb3e6aefddca10284c7e1cdc385b9 of algorithm-rls/lqpt:lqsrv-1.0.4-prod already exists on the destination registry, skip
2018-11-09T09:52:14Z [INFO] blob sha256:e152ae17623ec1f11a45f8837ac4600a7a38295e11ad94cca66d0edef67b7a55 of algorithm-rls/lqpt:lqsrv-1.0.4-prod already exists on the destination registry, skip
2018-11-09T09:52:14Z [INFO] blob sha256:c9e7c5a0aeefa70ee6b7227b8ac4ccc6508d7f6164d560be1d2acc29670dd446 of algorithm-rls/lqpt:lqsrv-1.0.4-prod already exists on the destination registry, skip
2018-11-09T09:52:14Z [INFO] blob sha256:653eaa6c1d2d3c8ea80dbe4b12e64120c62a95dcc63e23348f4f569579368c59 of algorithm-rls/lqpt:lqsrv-1.0.4-prod already exists on the destination registry, skip
2018-11-09T09:52:14Z [INFO] blob sha256:2bc130a59d4e448f0d7cb9490c3a19d034b6803bc3083751f18f8949714c524c of algorithm-rls/lqpt:lqsrv-1.0.4-prod already exists on the destination registry, skip
2018-11-09T09:52:14Z [INFO] manifest of algorithm-rls/lqpt:lqsrv-1.0.4-prod exists on the destination registry, skip manifest pushing
2018-11-09T09:52:14Z [INFO] manifest of algorithm-rls/lqpt:lqsrv_global-0.7.2 pulled successfully from source registry: sha256:c9d4afc0baea5742e6d6ad9f4861a4317b17ca02855d9b35b78d11cdcb9c92cf
2018-11-09T09:52:14Z [INFO] blob sha256:d55425e39d86865fe4c4faa58866af13c6a5a012c7d14eae532532c5190b68e4 of algorithm-rls/lqpt:lqsrv_global-0.7.2 already exists on the destination registry, skip
2018-11-09T09:52:14Z [INFO] blob sha256:1e76f742da490c8d7c921e811e5233def206e76683ee28d735397ec2231f131d of algorithm-rls/lqpt:lqsrv_global-0.7.2 already exists on the destination registry, skip
2018-11-09T09:52:14Z [INFO] blob sha256:15ad779f15c7efca616eecfc6d848d0c25d3c7aea4dc28e449912b5017814512 of algorithm-rls/lqpt:lqsrv_global-0.7.2 already exists on the destination registry, skip
2018-11-09T09:52:14Z [INFO] blob sha256:8aa6557b859ecc090eed6ae57af0cb3a74ce77cf9829dafa269872cc69f94bfc of algorithm-rls/lqpt:lqsrv_global-0.7.2 already exists on the destination registry, skip
2018-11-09T09:52:14Z [INFO] blob sha256:55b0422f0ff2e6159f93a99ac578b770d5e06b5acc720e7423f6168761f8c8ae of algorithm-rls/lqpt:lqsrv_global-0.7.2 already exists on the destination registry, skip
2018-11-09T09:52:14Z [INFO] blob sha256:edfece5caca755d865c6ce1491b8a055c98479f9f402ed44107d7cd23e164868 of algorithm-rls/lqpt:lqsrv_global-0.7.2 already exists on the destination registry, skip
2018-11-09T09:52:14Z [INFO] blob sha256:edfece5caca755d865c6ce1491b8a055c98479f9f402ed44107d7cd23e164868 of algorithm-rls/lqpt:lqsrv_global-0.7.2 already exists on the destination registry, skip
2018-11-09T09:52:14Z [INFO] manifest of algorithm-rls/lqpt:lqsrv_global-0.7.2 exists on the destination registry, skip manifest pushing
```