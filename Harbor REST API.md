# Harbor REST API

## 基于 Swagger 查看和测试 Harbor REST API

> 参考：[View and test Harbor REST API via Swagger](https://github.com/vmware/harbor/blob/master/docs/configure_swagger.md)

A Swagger file is provided for viewing and testing Harbor REST API.

- `swagger.yaml` under the docs directory
- Online Swagger Editor: http://editor.swagger.io/
- Local Swagger Testing: https://11.11.11.12/static/vendors/swagger/index.html
- When using Swagger to send REST requests to Harbor, you may alter the data of Harbor accidentally.


----------


![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/Harbor%20API%20-%201.png)

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/Harbor%20API%20-%202.png)

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/Harbor%20API%20-%203.png)

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/Harbor%20API%20-%204.png)


> 如下只列出了当前关注的部分接口

### GET /search

The Search endpoint returns information about the **projects** and **repositories** offered at **public** status or related to the **current logged in user**. The response includes the project and repository list in a proper display order.

| Name | Description |
| --- | --- |
| q (**required**) <br> string (query) | Search parameter for project and repository name. |

```
$ curl -X GET --header 'Accept: application/json' 'https://11.11.11.12/api/search?q=prj' -k
{
  "project": [
    {
      "project_id": 3,
      "owner_id": 1,
      "name": "prj2",
      "creation_time": "2017-10-24T03:06:41Z",
      "creation_time_str": "",
      "deleted": 0,
      "owner_name": "",
      "public": 1,
      "Togglable": false,
      "update_time": "2017-10-24T03:06:41Z",
      "current_user_role_id": 0,
      "repo_count": 0,
      "enable_content_trust": false,
      "prevent_vulnerable_images_from_running": false,
      "prevent_vulnerable_images_from_running_severity": "",
      "automatically_scan_images_on_push": false
    },
    {
      "project_id": 4,
      "owner_id": 1,
      "name": "prj3",
      "creation_time": "2017-10-24T07:24:14Z",
      "creation_time_str": "",
      "deleted": 0,
      "owner_name": "",
      "public": 1,
      "Togglable": false,
      "update_time": "2017-10-24T07:24:14Z",
      "current_user_role_id": 0,
      "repo_count": 0,
      "enable_content_trust": false,
      "prevent_vulnerable_images_from_running": false,
      "prevent_vulnerable_images_from_running_severity": "",
      "automatically_scan_images_on_push": false
    }
  ],
  "repository": []
}
```


### GET /projects

This endpoint returns all projects created by Harbor, and can be filtered by project name.


| Name | Description |
| --- | --- |
| name <br> string (query) | The name of project. |
| public <br> boolean (query) | The project is public or private. |
| owner <br> string (query) | The name of project owner. |
| page <br> integer (query) | The page nubmer, default is 1. |
| page_size <br> integer (query) | The size of per page, default is 10, maximum is 100. |

```
$ curl -X GET --header 'Accept: application/json' 'https://11.11.11.12/api/projects?name=prj&public=true&owner=admin&page=1&page_size=10' -k
[
  {
    "project_id": 3,
    "owner_id": 1,
    "name": "prj2",
    "creation_time": "2017-10-24T03:06:41Z",
    "creation_time_str": "",
    "deleted": 0,
    "owner_name": "",
    "public": 1,
    "Togglable": false,
    "update_time": "2017-10-24T03:06:41Z",
    "current_user_role_id": 0,
    "repo_count": 0,
    "enable_content_trust": false,
    "prevent_vulnerable_images_from_running": false,
    "prevent_vulnerable_images_from_running_severity": "",
    "automatically_scan_images_on_push": false
  },
  {
    "project_id": 4,
    "owner_id": 1,
    "name": "prj3",
    "creation_time": "2017-10-24T07:24:14Z",
    "creation_time_str": "",
    "deleted": 0,
    "owner_name": "",
    "public": 1,
    "Togglable": false,
    "update_time": "2017-10-24T07:24:14Z",
    "current_user_role_id": 0,
    "repo_count": 0,
    "enable_content_trust": false,
    "prevent_vulnerable_images_from_running": false,
    "prevent_vulnerable_images_from_running_severity": "",
    "automatically_scan_images_on_push": false
  }
]
```


### POST /projects

This endpoint is for user to create a new project.


```
[#61#root@ubuntu-1604 /opt/apps/harbor]$curl -X POST --header 'Content-Type: text/plain' --header 'Accept: text/plain' -d '{
  "project_name": "prj5",
  "public": 0,
  "enable_content_trust": false,
  "prevent_vulnerable_images_from_running": false,
  "prevent_vulnerable_images_from_running_severity": "string",
  "automatically_scan_images_on_push": true
}' 'https://11.11.11.12/api/projects' -k -i
HTTP/1.1 401 Unauthorized         -- User need to log in first.
Server: nginx/1.11.13
Date: Tue, 24 Oct 2017 07:53:32 GMT
Content-Type: text/plain; charset=utf-8
Content-Length: 1
Connection: keep-alive
Set-Cookie: beegosessionID=4008dd9467bd42c155017ca947d3ac4c; Path=/; secure; HttpOnly
X-Content-Type-Options: nosniff

[#62#root@ubuntu-1604 /opt/apps/harbor]$
[#62#root@ubuntu-1604 /opt/apps/harbor]$
[#62#root@ubuntu-1604 /opt/apps/harbor]$curl -X POST --header 'Content-Type: text/plain' --header 'Accept: text/plain' -d '{
  "project_name": "prj5",
  "public": 0,
  "enable_content_trust": false,
  "prevent_vulnerable_images_from_running": false,
  "prevent_vulnerable_images_from_running_severity": "string",
  "automatically_scan_images_on_push": true
}' 'https://11.11.11.12/api/projects' -k -i -u admin:Harbor12345
HTTP/1.1 201 Created           -- Project created successfully.
Server: nginx/1.11.13
Date: Tue, 24 Oct 2017 07:55:25 GMT
Content-Type: text/plain; charset=utf-8
Content-Length: 0
Connection: keep-alive
Location: /api/projects/6
Set-Cookie: beegosessionID=1b96afa9f5050c0620d38a93c6e741dd; Path=/; secure; HttpOnly

[#63#root@ubuntu-1604 /opt/apps/harbor]$



[#63#root@ubuntu-1604 /opt/apps/harbor]$curl -X POST --header 'Content-Type: text/plain' --header 'Accept: text/plain' -d '{
  "project_name": "prj5",
  "public": 0,
  "enable_content_trust": false,
  "prevent_vulnerable_images_from_running": false,
  "prevent_vulnerable_images_from_running_severity": "string",
  "automatically_scan_images_on_push": true
}' 'https://11.11.11.12/api/projects' -k -i -u admin:Harbor12345
HTTP/1.1 409 Conflict        -- Project name already exists.
Server: nginx/1.11.13
Date: Tue, 24 Oct 2017 07:57:15 GMT
Content-Type: text/plain; charset=utf-8
Content-Length: 1
Connection: keep-alive
Set-Cookie: beegosessionID=d60977bff452c081462d7f496686cb26; Path=/; secure; HttpOnly
X-Content-Type-Options: nosniff


[#64#root@ubuntu-1604 /opt/apps/harbor]$
```



### GET /projects/{project_id}

This endpoint returns specific project information by project ID.


```
[#66#root@ubuntu-1604 /opt/apps/harbor]$curl -X GET --header 'Accept: text/plain' 'https://11.11.11.12/api/projects/0' -k -i
HTTP/1.1 400 Bad Request
Server: nginx/1.11.13
Date: Tue, 24 Oct 2017 08:03:04 GMT
Content-Type: text/plain; charset=utf-8
Content-Length: 22
Connection: keep-alive
Set-Cookie: beegosessionID=543182574d93b00a77d0646b015def44; Path=/; secure; HttpOnly
X-Content-Type-Options: nosniff

invalid project ID: 0
[#67#root@ubuntu-1604 /opt/apps/harbor]$
[#67#root@ubuntu-1604 /opt/apps/harbor]$curl -X GET --header 'Accept: text/plain' 'https://11.11.11.12/api/projects/1' -k -i
HTTP/1.1 200 OK
Server: nginx/1.11.13
Date: Tue, 24 Oct 2017 08:03:09 GMT
Content-Type: application/json; charset=utf-8
Content-Length: 478
Connection: keep-alive
Set-Cookie: beegosessionID=5d135eb1970916409ed2918da8f8fb84; Path=/; secure; HttpOnly

{
  "project_id": 1,
  "owner_id": 1,
  "name": "library",
  "creation_time": "2017-10-23T08:16:06Z",
  "creation_time_str": "",
  "deleted": 0,
  "owner_name": "",
  "public": 1,
  "Togglable": false,
  "update_time": "2017-10-23T08:16:06Z",
  "current_user_role_id": 0,
  "repo_count": 0,
  "enable_content_trust": false,
  "prevent_vulnerable_images_from_running": false,
  "prevent_vulnerable_images_from_running_severity": "",
  "automatically_scan_images_on_push": false
}[#68#root@ubuntu-1604 /opt/apps/harbor]$
[#68#root@ubuntu-1604 /opt/apps/harbor]$curl -X GET --header 'Accept: text/plain' 'https://11.11.11.12/api/projects/2' -k -i
HTTP/1.1 401 Unauthorized
Server: nginx/1.11.13
Date: Tue, 24 Oct 2017 08:03:13 GMT
Content-Type: text/plain; charset=utf-8
Content-Length: 1
Connection: keep-alive
Set-Cookie: beegosessionID=2d5099fcfaa9c258f2d123d96674e295; Path=/; secure; HttpOnly
X-Content-Type-Options: nosniff


[#69#root@ubuntu-1604 /opt/apps/harbor]$curl -X GET --header 'Accept: text/plain' 'https://11.11.11.12/api/projects/3' -k -i
HTTP/1.1 200 OK
Server: nginx/1.11.13
Date: Tue, 24 Oct 2017 08:03:26 GMT
Content-Type: application/json; charset=utf-8
Content-Length: 475
Connection: keep-alive
Set-Cookie: beegosessionID=29aea3a1250dfd9cb4a31af89580a6ef; Path=/; secure; HttpOnly

{
  "project_id": 3,
  "owner_id": 1,
  "name": "prj2",
  "creation_time": "2017-10-24T03:06:41Z",
  "creation_time_str": "",
  "deleted": 0,
  "owner_name": "",
  "public": 1,
  "Togglable": false,
  "update_time": "2017-10-24T03:06:41Z",
  "current_user_role_id": 0,
  "repo_count": 0,
  "enable_content_trust": false,
  "prevent_vulnerable_images_from_running": false,
  "prevent_vulnerable_images_from_running_severity": "",
  "automatically_scan_images_on_push": false
}[#70#root@ubuntu-1604 /opt/apps/harbor]$
[#70#root@ubuntu-1604 /opt/apps/harbor]$curl -X GET --header 'Accept: text/plain' 'https://11.11.11.12/api/projects/4' -k -i
HTTP/1.1 200 OK
Server: nginx/1.11.13
Date: Tue, 24 Oct 2017 08:03:35 GMT
Content-Type: application/json; charset=utf-8
Content-Length: 475
Connection: keep-alive
Set-Cookie: beegosessionID=c336115ebbb88ea9c07e1c85709e88e0; Path=/; secure; HttpOnly

{
  "project_id": 4,
  "owner_id": 1,
  "name": "prj3",
  "creation_time": "2017-10-24T07:24:14Z",
  "creation_time_str": "",
  "deleted": 0,
  "owner_name": "",
  "public": 1,
  "Togglable": false,
  "update_time": "2017-10-24T07:24:14Z",
  "current_user_role_id": 0,
  "repo_count": 0,
  "enable_content_trust": false,
  "prevent_vulnerable_images_from_running": false,
  "prevent_vulnerable_images_from_running_severity": "",
  "automatically_scan_images_on_push": false
}[#71#root@ubuntu-1604 /opt/apps/harbor]$
[#71#root@ubuntu-1604 /opt/apps/harbor]$
[#71#root@ubuntu-1604 /opt/apps/harbor]$curl -X GET --header 'Accept: text/plain' 'https://11.11.11.12/api/projects/5' -k -i
HTTP/1.1 401 Unauthorized
Server: nginx/1.11.13
Date: Tue, 24 Oct 2017 08:04:04 GMT
Content-Type: text/plain; charset=utf-8
Content-Length: 1
Connection: keep-alive
Set-Cookie: beegosessionID=45b22a90d34f72ce6e3c1835a9c7172c; Path=/; secure; HttpOnly
X-Content-Type-Options: nosniff


[#72#root@ubuntu-1604 /opt/apps/harbor]$
[#72#root@ubuntu-1604 /opt/apps/harbor]$curl -X GET --header 'Accept: text/plain' 'https://11.11.11.12/api/projects/6' -k -i
HTTP/1.1 401 Unauthorized
Server: nginx/1.11.13
Date: Tue, 24 Oct 2017 08:04:08 GMT
Content-Type: text/plain; charset=utf-8
Content-Length: 1
Connection: keep-alive
Set-Cookie: beegosessionID=04bb3ba6801d0799b5acb0defa88ce21; Path=/; secure; HttpOnly
X-Content-Type-Options: nosniff


[#73#root@ubuntu-1604 /opt/apps/harbor]$
[#73#root@ubuntu-1604 /opt/apps/harbor]$curl -X GET --header 'Accept: text/plain' 'https://11.11.11.12/api/projects/7' -k -i
HTTP/1.1 404 Not Found
Server: nginx/1.11.13
Date: Tue, 24 Oct 2017 08:04:11 GMT
Content-Type: text/plain; charset=utf-8
Content-Length: 20
Connection: keep-alive
Set-Cookie: beegosessionID=6ec9818ece8a5857b1ebc0f8b41c6123; Path=/; secure; HttpOnly
X-Content-Type-Options: nosniff

project 7 not found
[#74#root@ubuntu-1604 /opt/apps/harbor]$
[#74#root@ubuntu-1604 /opt/apps/harbor]$curl -X GET --header 'Accept: text/plain' 'https://11.11.11.12/api/projects/6' -k -i -u admin:Harbor12345
HTTP/1.1 200 OK
Server: nginx/1.11.13
Date: Tue, 24 Oct 2017 08:04:31 GMT
Content-Type: application/json; charset=utf-8
Content-Length: 475
Connection: keep-alive
Set-Cookie: beegosessionID=d1deeb50d97417ec10e657fcb880ccb4; Path=/; secure; HttpOnly

{
  "project_id": 6,
  "owner_id": 1,
  "name": "prj5",
  "creation_time": "2017-10-24T07:55:25Z",
  "creation_time_str": "",
  "deleted": 0,
  "owner_name": "",
  "public": 0,
  "Togglable": false,
  "update_time": "2017-10-24T07:55:25Z",
  "current_user_role_id": 0,
  "repo_count": 0,
  "enable_content_trust": false,
  "prevent_vulnerable_images_from_running": false,
  "prevent_vulnerable_images_from_running_severity": "",
  "automatically_scan_images_on_push": false
}[#75#root@ubuntu-1604 /opt/apps/harbor]$
```

### GET /statistics

This endpoint is aimed to statistic all of the **projects number** and **repositories number** relevant to the **logined user**, also the public projects number and repositories number. If the user is admin, he can also get total projects number and total repositories number.


```
[#78#root@ubuntu-1604 /opt/apps/harbor]$curl -X GET --header 'Accept: application/json' 'https://11.11.11.12/api/statistics' -k

[#79#root@ubuntu-1604 /opt/apps/harbor]$
[#79#root@ubuntu-1604 /opt/apps/harbor]$curl -X GET --header 'Accept: application/json' 'https://11.11.11.12/api/statistics' -k -u admin:Harbor12345
{
  "private_project_count": 3,
  "private_repo_count": 1,
  "public_project_count": 3,
  "public_repo_count": 0,
  "total_project_count": 6,
  "total_repo_count": 1
}[#80#root@ubuntu-1604 /opt/apps/harbor]$
```

### GET /repositories


This endpoint let user search repositories accompanying with relevant project ID and repo name.

| Name | Description |
| --- | --- |
| project_id (**required**) <br> integer (query) | Relevant project ID. |
| q <br> string (query) | Repo name for filtering results. |
| page <br> integer (query) | The page nubmer, default is 1. |
| page_size <br> integer (query) | The size of per page, default is 10, maximum is 100. |


```
[#95#root@ubuntu-1604 ~]$curl -X GET --header 'Accept: application/json' 'https://11.11.11.12/api/repositories?project_id=2&q=prj&page=1&page_size=10' -k -i
HTTP/1.1 401 Unauthorized
Server: nginx/1.11.13
Date: Tue, 24 Oct 2017 09:30:48 GMT
Content-Type: text/plain; charset=utf-8
Content-Length: 1
Connection: keep-alive
Set-Cookie: beegosessionID=19a08cb1795d2ccb34e7b2cdd565d186; Path=/; secure; HttpOnly
X-Content-Type-Options: nosniff


[#96#root@ubuntu-1604 ~]$curl -X GET --header 'Accept: application/json' 'https://11.11.11.12/api/repositories?project_id=2&q=prj&page=1&page_size=10' -k -i -u admin:Harbor12345
HTTP/1.1 200 OK
Server: nginx/1.11.13
Date: Tue, 24 Oct 2017 09:31:00 GMT
Content-Type: application/json; charset=utf-8
Content-Length: 250
Connection: keep-alive
Set-Cookie: beegosessionID=3811e8d90855cf4c712e8dc6a02be082; Path=/; secure; HttpOnly
X-Total-Count: 1

[
  {
    "id": 1,
    "name": "prj1/hello-world",
    "project_id": 2,
    "description": "",
    "pull_count": 0,
    "star_count": 0,
    "tags_count": 3,
    "creation_time": "2017-10-24T06:37:54Z",
    "update_time": "2017-10-24T06:37:54Z"
  }
][#97#root@ubuntu-1604 ~]$
```

### GET /repositories/{repo_name}/tags/{tag}

Get the tag of the repository.

### GET /repositories/{repo_name}/tags

Get tags of a relevant repository.

### GET /repositories/{repo_name}/tags/{tag}/manifest

Get manifests of a relevant repository.

### GET /repositories/{repo_name}/tags/{tag}/vulnerability/details

Get vulnerability details of the image.

### GET /repositories/{repo_name}/signatures

Get signature information of a repository

### GET /repositories/top

Get public repositories which are accessed most.

### GET /logs

Get recent logs of the projects which the user is a member of

### GET /jobs/replication

List filters jobs according to the policy and repository

### GET /systeminfo

Get general system info

### GET /systeminfo/volumes

Get system volume info (total/free size).

### GET /systeminfo/getcert

Get default root certificate under OVA deployment.

### GET /configurations

Get system configurations.

