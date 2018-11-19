# github-release 项目研究

| key | value |
| -- | -- |
| 项目名称 | github-release |
| 地址 | https://github.com/aktau/github-release  |
| 实现语言 | golang |
| 日期 | 2018-11-19 |
| 一句话描述 | CLI 应用，用于创建和编辑 Github 上的 releases ，以及上传 artifacts |
| 详细说明 | <br> 1. create and delete releases of your projects on Github. <br> 2. allow you to attach files to those releases, <br> 3. Interacts with the [github releases API](http://developer.github.com/v3/repos/releases). |
| 项目依赖 | <br> 1. github.com/tomnomnom/linkheader (**Golang HTTP Link header parser**) <br> 2. github.com/dustin/go-humanize (**formatters for units to human friendly sizes**) <br> 3. github.com/voxelbrain/goptions (**A flexible parser for command line options**) |


----------

## 试用

- 安装

```
go get -v github.com/aktau/github-release
```

- [设置 GITHUB_TOKEN](https://help.github.com/articles/creating-a-personal-access-token-for-the-command-line/)

```
export GITHUB_TOKEN=6b1c019a5708a0b55dff66260742a135f05e96fc
```

- 帮助

```
[#13#root@ubuntu-1604 ~]$github-release --help
Usage: github-release [global options] <verb> [verb options]

Global options:
        -h, --help           Show this help
        -v, --verbose        Be verbose
        -q, --quiet          Do not print anything, even errors (except if --verbose is specified)
            --version        Print version

Verbs:
    delete:
        -s, --security-token Github token (required if $GITHUB_TOKEN not set)
        -u, --user           Github repo user or organisation (required if $GITHUB_USER not set)
        -r, --repo           Github repo (required if $GITHUB_REPO not set)
        -t, --tag            Git tag of release to delete (*)
    download:
        -s, --security-token Github token ($GITHUB_TOKEN if set). required if repo is private.
        -u, --user           Github repo user or organisation (required if $GITHUB_USER not set)
        -r, --repo           Github repo (required if $GITHUB_REPO not set)
        -l, --latest         Download latest release (required if tag is not specified)
        -t, --tag            Git tag to download from (required if latest is not specified) (*)
        -n, --name           Name of the file (*)
    edit:
        -s, --security-token Github token (required if $GITHUB_TOKEN not set)
        -u, --user           Github repo user or organisation (required if $GITHUB_USER not set)
        -r, --repo           Github repo (required if $GITHUB_REPO not set)
        -t, --tag            Git tag to edit the release of (*)
        -n, --name           New name of the release (defaults to tag)
        -d, --description    New release description, use - for reading a description from stdin (defaults to tag)
            --draft          The release is a draft
        -p, --pre-release    The release is a pre-release
    info:
        -s, --security-token Github token ($GITHUB_TOKEN if set). required if repo is private.
        -u, --user           Github repo user or organisation (required if $GITHUB_USER not set)
        -r, --repo           Github repo (required if $GITHUB_REPO not set)
        -t, --tag            Git tag to query (optional)
        -j, --json           Emit info as JSON instead of text
    release:
        -s, --security-token Github token (required if $GITHUB_TOKEN not set)
        -u, --user           Github repo user or organisation (required if $GITHUB_USER not set)
        -r, --repo           Github repo (required if $GITHUB_REPO not set)
        -t, --tag            Git tag to create a release from (*)
        -n, --name           Name of the release (defaults to tag)
        -d, --description    Release description, use - for reading a description from stdin (defaults to tag)
        -c, --target         Commit SHA or branch to create release of (defaults to the repository default branch)
            --draft          The release is a draft
        -p, --pre-release    The release is a pre-release
    upload:
        -s, --security-token Github token (required if $GITHUB_TOKEN not set)
        -u, --user           Github repo user or organisation (required if $GITHUB_USER not set)
        -r, --repo           Github repo (required if $GITHUB_REPO not set)
        -t, --tag            Git tag to upload to (*)
        -n, --name           Name of the file (*)
        -l, --label          Label (description) of the file
        -f, --file           File to upload (use - for stdin) (*)
        -R, --replace        Replace asset with same name if it already exists (WARNING: not atomic, failure to upload will remove the original asset too)
[#14#root@ubuntu-1604 ~]$
```

- 版本

```
[#14#root@ubuntu-1604 ~]$github-release --version
github-release v0.7.2
[#15#root@ubuntu-1604 ~]$
```

- 基于 CLI 调用的正常返回

```
[#18#root@ubuntu-1604 ~]$github-release info -u moooofly -r harbor-go-client
tags:
- v1.1.2 (commit: https://api.github.com/repos/moooofly/harbor-go-client/commits/6e8e5ec302ba26edc0dbb37d7565ca04a3867177)
- v1.1.1 (commit: https://api.github.com/repos/moooofly/harbor-go-client/commits/85e7170a54f09de36f3166b928b3b3f84adfb7ef)
- v1.1.0 (commit: https://api.github.com/repos/moooofly/harbor-go-client/commits/3136284512ce94f3de71fb012d742c1fc4b0c30f)
- v1.0.0 (commit: https://api.github.com/repos/moooofly/harbor-go-client/commits/3d851c259530d41662e12b1eaa0a7a4925b1464e)
- v0.9.6 (commit: https://api.github.com/repos/moooofly/harbor-go-client/commits/9c400e9f1785236a92f06450c7cd2443f688405a)
- v0.9.5 (commit: https://api.github.com/repos/moooofly/harbor-go-client/commits/5dd4268eff2fd75417ec4ddccbea9c8dc539d4ff)
- v0.9.4 (commit: https://api.github.com/repos/moooofly/harbor-go-client/commits/eaa6c2b6d2c11d060baaf3ff75d48b1470a71ec0)
- v0.9.3 (commit: https://api.github.com/repos/moooofly/harbor-go-client/commits/bcdf9e38f0adafbb9f0dcee5a53347737307adf4)
- v0.9.2 (commit: https://api.github.com/repos/moooofly/harbor-go-client/commits/6e72c76af7e1c37b79d8db06fa9dfa78fef20fd9)
- v0.9.1 (commit: https://api.github.com/repos/moooofly/harbor-go-client/commits/360bc287242c500153f5c7349ca91c07ca53c6bd)
- v0.9.0 (commit: https://api.github.com/repos/moooofly/harbor-go-client/commits/5752a1674d5a1f2262c0e04e8206328e0504d23b)
- v0.8.0 (commit: https://api.github.com/repos/moooofly/harbor-go-client/commits/aecbcfa3570639426ec61382764220b17b80633f)
- v0.7.0 (commit: https://api.github.com/repos/moooofly/harbor-go-client/commits/baa8fe9b9a252959bbc87fd2b3550da88c4fc6f6)
- v0.6.0 (commit: https://api.github.com/repos/moooofly/harbor-go-client/commits/c0568b6a0a35228c71203dea04018dd296b14f23)
- v0.5.0 (commit: https://api.github.com/repos/moooofly/harbor-go-client/commits/d473c2c6a53c93971bbe901bec025ad28e147f6c)
- v0.4.0 (commit: https://api.github.com/repos/moooofly/harbor-go-client/commits/baed9807f4e0c1fc248f9ce3ff464d4b36dcb71e)
- v0.3.0 (commit: https://api.github.com/repos/moooofly/harbor-go-client/commits/919a7c36dd046bf2fd3afc9aa53ee86e95695c1f)
- v0.2.0 (commit: https://api.github.com/repos/moooofly/harbor-go-client/commits/ad89b5ff5abbd974bafd45b7ee644dc32b2427e6)
- v0.1.0 (commit: https://api.github.com/repos/moooofly/harbor-go-client/commits/07ae5171ca1dd89d17f897cd2d66dcc2c80d490f)
releases:
- v1.1.1, name: 'v1.1.1', description: '# Changelog

## Features

* add --dry-run to rp_tags sub-command. fix [#11](https://github.com/moooofly/harbor-go-client/issues/11) ([5fd4f39](https://github.com/moooofly/harbor-go-client/commit/5fd4f39))
', id: 13772700, tagged: 26/10/2018 at 02:46, published: 01/11/2018 at 09:24, draft: ✗, prerelease: ✗
  - artifact: harborctl-1.1.1.darwin-amd64.tar.gz, downloads: 5, state: uploaded, type: application/x-gzip, size: 4.7 MB, id: 9505961
  - artifact: harborctl-1.1.1.linux-amd64.tar.gz, downloads: 4, state: uploaded, type: application/x-gzip, size: 4.7 MB, id: 9505962
- v1.0.0, name: 'v1.0.0', description: '# Changelog

## Features

* **replication api:** add replications trigger API ([0753d6c](https://github.com/moooofly/harbor-go-client/commit/0753d6c))
* add syncregistry and email_ping APIs ([2b18258](https://github.com/moooofly/harbor-go-client/commit/2b18258))
* **usergroup api:** add usergroups APIs ([47ef550](https://github.com/moooofly/harbor-go-client/commit/47ef550))
* support both linux and darwin platform compilation ([a665e0f](https://github.com/moooofly/harbor-go-client/commit/a665e0f))
* support darwin platform, fix [#1](https://github.com/moooofly/harbor-go-client/issues/1) ([c671c20](https://github.com/moooofly/harbor-go-client/commit/c671c20))
* **label api:** add labels APIs ([603570c](https://github.com/moooofly/harbor-go-client/commit/603570c))
* **policy api:** add all policy APIs ([53e2193](https://github.com/moooofly/harbor-go-client/commit/53e2193))
* **project api:** add prj_member_update/prj_member_get/prj_member_del/prj_member_create/prj_members ([fea77d2](https://github.com/moooofly/harbor-go-client/commit/fea77d2))
* **project api:** add prj_metadata_update_by_name/prj_metadata_get_by_name/prj_metadata_del_by_name ([0020cec](https://github.com/moooofly/harbor-go-client/commit/0020cec))
* **project api:** add prj_metadata_add/prj_metadata_get/prj_logs_get/prj_update APIs ([5770328](https://github.com/moooofly/harbor-go-client/commit/5770328))
* **repository api:** add repo_signature_get/repo_image_manifests_get APIs ([95d322d](https://github.com/moooofly/harbor-go-client/commit/95d322d))
* **repository api:** add repo_image_label_del api ([b5b152a](https://github.com/moooofly/harbor-go-client/commit/b5b152a))
* **repository api:** add repo_image_label_add api ([01f7885](https://github.com/moooofly/harbor-go-client/commit/01f7885))
* **repository api:** add repo_image_labels_get/repo_label_del APIs ([54e02be](https://github.com/moooofly/harbor-go-client/commit/54e02be))
* **repository api:** add repo_label_add/repo_labels_get/repo_desp_update APIs ([c12b012](https://github.com/moooofly/harbor-go-client/commit/c12b012))
* add fake *_test.go ([0f0c6f4](https://github.com/moooofly/harbor-go-client/commit/0f0c6f4))
* **api:** add api/doc.go for golang docs ([928c691](https://github.com/moooofly/harbor-go-client/commit/928c691))

## Bug Fixes

* **regression test script:** fix wrong use of printf ([c3ffb03](https://github.com/moooofly/harbor-go-client/commit/c3ffb03))
', id: 13634946, tagged: 24/10/2018 at 15:10, published: 24/10/2018 at 15:29, draft: ✗, prerelease: ✗
  - artifact: harborctl-1.0.0.darwin-amd64.tar.gz, downloads: 1, state: uploaded, type: application/x-gzip, size: 4.7 MB, id: 9393688
  - artifact: harborctl-1.0.0.linux-amd64.tar.gz, downloads: 0, state: uploaded, type: application/x-gzip, size: 4.7 MB, id: 9393689
- v0.9.0, name: 'v0.9.0', description: '# Changelog

* d8b4813 - feat: add All users APIs', id: 12051523, tagged: 23/07/2018 at 09:28, published: 23/07/2018 at 09:35, draft: ✗, prerelease: ✗
  - artifact: v0.9.0-bin.tar.gz, downloads: 22, state: uploaded, type: application/x-gzip, size: 3.0 MB, id: 7972240
- v0.8.0, name: 'v0.8.0', description: '# Changelog

* 3d5ce7e - Fix compatibility issue derived from harbor v1.3.0

**NOTE**: harbor-go-client v0.8.0 is compatible with harbor v1.3.0 and later. see #2 ', id: 11870625, tagged: 11/07/2018 at 07:22, published: 11/07/2018 at 07:29, draft: ✗, prerelease: ✗
  - artifact: v0.8.0-bin.tar.gz, downloads: 5, state: uploaded, type: application/x-gzip, size: 3.0 MB, id: 7833845
- v0.7.0, name: 'v0.7.0', description: '# Changelog

* 477af58 - Merge branch 'feature/makefile_optimize' into develop
* 1bc2d9f - feat: optimize *.sh according to shellcheck
* 49b717d - feat: optimize 'make test' cmd
* c1029c4 - feat: add Dockerfile and add 'make docker' cmd
* 69932e0 - chore: format adjustment on regression_test.sh

------

**NOTE**: harbor-go-client v0.7.0 and before is compatible with harbor v1.2.2 and before. see #2 ', id: 11808264, tagged: 06/07/2018 at 09:32, published: 06/07/2018 at 09:41, draft: ✗, prerelease: ✗
  - artifact: v0.7.0-bin.tar.gz, downloads: 22, state: uploaded, type: application/x-gzip, size: 2.8 MB, id: 7783730
- v0.6.0, name: 'v0.6.0', description: '# Changelog

* c0568b6 - feat: integrate packing function into Makefile
* b1b40e0 - Merge branch 'feature/add_jobs_related_api' into develop
* ea47987 - feat: make version info generating automatically
* f093d71 - feat: optimize api/logs.go on values of 'operation'
* ded0a81 - feat: add jobs related APIs
* 3b18281 - chore: update comment
* 25be99a - feat: do some adjustments for conf/
* 3691179 - chore: move scripts/*.yaml to conf/*.yaml', id: 11671107, tagged: 27/06/2018 at 12:02, published: 27/06/2018 at 12:13, draft: ✗, prerelease: ✗
  - artifact: v0.6.0-bin.tar.gz, downloads: 3, state: uploaded, type: application/x-gzip, size: 2.8 MB, id: 7682458
- v0.5.0, name: 'v0.5.0', description: '# Changelog

* c58c652 - Merge branch 'feature/prj_structure_change'
* e3a5245 - update `scripts/binPack.sh` for easy packaging
* b0ee29a - chore: update `regression_test.sh` and `rp_repos_simulation.sh`
* 0f64503 - rename: `rp_repos.sh` -> `rp_repos_simulation.sh`
* 60914ec - update: moving *.sh into scripts/
* 1180efa - rename: `test.sh` -> `regression_test.sh`
* 00f1611 - add issue templates
* aa79c4a - add `CHANGELOG.md`
* 8624e9f - add version info output', id: 11604781, tagged: 22/06/2018 at 11:31, published: 22/06/2018 at 11:43, draft: ✗, prerelease: ✗
  - artifact: v0.5.0-bin.tar.gz, downloads: 0, state: uploaded, type: application/x-gzip, size: 2.9 MB, id: 7628672
- v0.4.0, name: 'v0.4.0', description: '# Changelog

* 86ac060 - chore: update doc
* 34ee6a6 - feat: add doc '关于清理策略的讨论'
* 82740c7 - chore: optimize log output', id: 11564923, tagged: 20/06/2018 at 09:44, published: 20/06/2018 at 09:50, draft: ✗, prerelease: ✗
  - artifact: v0.4.0-bin.tar.gz, downloads: 2, state: uploaded, type: application/x-gzip, size: 2.9 MB, id: 7599233
- v0.3.0, name: 'v0.3.0', description: '# Changelog

* 5d9397b - Merge branch 'feature/fix_heapsort_bug_in_rp_tags'
* f2d501a - chore: optimize log output
* c2a3427 - fix: **change heapsort of tags from maxheap to minheap**
* bd5513c - update `.gitignore`
* 286ec68 - chore: add `binPack.sh`
* 1a47e33 - fix: change specific ip address to `localhost`', id: 11543467, tagged: 19/06/2018 at 08:09, published: 19/06/2018 at 08:45, draft: ✗, prerelease: ✗
  - artifact: v0.3.0-bin.tar.gz, downloads: 1, state: uploaded, type: application/x-gzip, size: 2.9 MB, id: 7585045
- v0.2.0, name: 'v0.2.0', description: '# Changelog

* 41c7394 - update `.gitignore`
* 005c46f - update README.md
* 002a3ab - update LICENSE
* 6674495 - add 'MIT LICENSE' badge
* eb94993 - add 'Build Status' badge
* b4c4696 - add `misspell` support
* f3b6c33 - fix: add `golint` and `gometalinter` in `.travis.yml`
* 1974b63 - some improvements for Travis CI', id: 11453825, tagged: 13/06/2018 at 03:35, published: 13/06/2018 at 06:01, draft: ✗, prerelease: ✗
  - artifact: v0.2.0-bin.tar.gz, downloads: 4, state: uploaded, type: application/x-gzip, size: 2.9 MB, id: 7528723
- v0.1.0, name: 'v0.1.0', description: 'This is the first release on GitHub after migrating from GitLab.

# Changelog

* 07ae517 - add `.travis.yml`
* 67af84f - add 'Go Report Card'
* e5be67b - fix typo
* 252d7e3 - update dependencies managed by `glide`
* 11af3bd - remove `harbor-go-client` executable
* b6e8ca0 - change the default value of `dstip` to `localhost` in config.yaml
* 1c95b68 - add `.gitignore`
* 1a46a0b - move gitlab to github
* 2fa21cb - Initial commit', id: 11453697, tagged: 12/06/2018 at 06:36, published: 13/06/2018 at 05:42, draft: ✗, prerelease: ✗
[#19#root@ubuntu-1604 ~]$
[#19#root@ubuntu-1604 ~]$
[#19#root@ubuntu-1604 ~]$
[#19#root@ubuntu-1604 ~]$github-release info -u moooofly -r harbor-go-client -t v1.1.1 -j
{
    "Tags": [
        {
            "name": "v1.1.1",
            "commit": {
                "sha": "85e7170a54f09de36f3166b928b3b3f84adfb7ef",
                "url": "https://api.github.com/repos/moooofly/harbor-go-client/commits/85e7170a54f09de36f3166b928b3b3f84adfb7ef"
            },
            "zipball_url": "https://api.github.com/repos/moooofly/harbor-go-client/zipball/v1.1.1",
            "tarball_url": "https://api.github.com/repos/moooofly/harbor-go-client/tarball/v1.1.1"
        }
    ],
    "Releases": [
        {
            "url": "https://api.github.com/repos/moooofly/harbor-go-client/releases/13772700",
            "html_url": "https://github.com/moooofly/harbor-go-client/releases/tag/v1.1.1",
            "upload_url": "https://uploads.github.com/repos/moooofly/harbor-go-client/releases/13772700/assets{?name,label}",
            "id": 13772700,
            "name": "v1.1.1",
            "body": "# Changelog\r\n\r\n## Features\r\n\r\n* add --dry-run to rp_tags sub-command. fix [#11](https://github.com/moooofly/harbor-go-client/issues/11) ([5fd4f39](https://github.com/moooofly/harbor-go-client/commit/5fd4f39))\r\n",
            "tag_name": "v1.1.1",
            "draft": false,
            "prerelease": false,
            "created_at": "2018-10-26T02:46:17Z",
            "published_at": "2018-11-01T09:24:43Z",
            "assets": [
                {
                    "url": "https://api.github.com/repos/moooofly/harbor-go-client/releases/assets/9505961",
                    "id": 9505961,
                    "name": "harborctl-1.1.1.darwin-amd64.tar.gz",
                    "content_type": "application/x-gzip",
                    "state": "uploaded",
                    "size": 4708728,
                    "download_count": 5,
                    "created_at": "2018-11-01T09:23:26Z",
                    "published_at": "0001-01-01T00:00:00Z"
                },
                {
                    "url": "https://api.github.com/repos/moooofly/harbor-go-client/releases/assets/9505962",
                    "id": 9505962,
                    "name": "harborctl-1.1.1.linux-amd64.tar.gz",
                    "content_type": "application/x-gzip",
                    "state": "uploaded",
                    "size": 4682662,
                    "download_count": 4,
                    "created_at": "2018-11-01T09:23:27Z",
                    "published_at": "0001-01-01T00:00:00Z"
                }
            ]
        }
    ]
}
[#20#root@ubuntu-1604 ~]$
```

- 在 web 上直接访问 https://api.github.com/repos/moooofly/harbor-go-client/tags?per_page=100 时的正常返回

```
[
  {
    "name": "v1.1.2",
    "zipball_url": "https://api.github.com/repos/moooofly/harbor-go-client/zipball/v1.1.2",
    "tarball_url": "https://api.github.com/repos/moooofly/harbor-go-client/tarball/v1.1.2",
    "commit": {
      "sha": "6e8e5ec302ba26edc0dbb37d7565ca04a3867177",
      "url": "https://api.github.com/repos/moooofly/harbor-go-client/commits/6e8e5ec302ba26edc0dbb37d7565ca04a3867177"
    },
    "node_id": "MDM6UmVmMTM2NTcxOTAzOnYxLjEuMg=="
  },
  {
    "name": "v1.1.1",
    "zipball_url": "https://api.github.com/repos/moooofly/harbor-go-client/zipball/v1.1.1",
    "tarball_url": "https://api.github.com/repos/moooofly/harbor-go-client/tarball/v1.1.1",
    "commit": {
      "sha": "85e7170a54f09de36f3166b928b3b3f84adfb7ef",
      "url": "https://api.github.com/repos/moooofly/harbor-go-client/commits/85e7170a54f09de36f3166b928b3b3f84adfb7ef"
    },
    "node_id": "MDM6UmVmMTM2NTcxOTAzOnYxLjEuMQ=="
  },
  {
    "name": "v1.1.0",
    "zipball_url": "https://api.github.com/repos/moooofly/harbor-go-client/zipball/v1.1.0",
    "tarball_url": "https://api.github.com/repos/moooofly/harbor-go-client/tarball/v1.1.0",
    "commit": {
      "sha": "3136284512ce94f3de71fb012d742c1fc4b0c30f",
      "url": "https://api.github.com/repos/moooofly/harbor-go-client/commits/3136284512ce94f3de71fb012d742c1fc4b0c30f"
    },
    "node_id": "MDM6UmVmMTM2NTcxOTAzOnYxLjEuMA=="
  },
  {
    "name": "v1.0.0",
    "zipball_url": "https://api.github.com/repos/moooofly/harbor-go-client/zipball/v1.0.0",
    "tarball_url": "https://api.github.com/repos/moooofly/harbor-go-client/tarball/v1.0.0",
    "commit": {
      "sha": "3d851c259530d41662e12b1eaa0a7a4925b1464e",
      "url": "https://api.github.com/repos/moooofly/harbor-go-client/commits/3d851c259530d41662e12b1eaa0a7a4925b1464e"
    },
    "node_id": "MDM6UmVmMTM2NTcxOTAzOnYxLjAuMA=="
  },
  {
    "name": "v0.9.6",
    "zipball_url": "https://api.github.com/repos/moooofly/harbor-go-client/zipball/v0.9.6",
    "tarball_url": "https://api.github.com/repos/moooofly/harbor-go-client/tarball/v0.9.6",
    "commit": {
      "sha": "9c400e9f1785236a92f06450c7cd2443f688405a",
      "url": "https://api.github.com/repos/moooofly/harbor-go-client/commits/9c400e9f1785236a92f06450c7cd2443f688405a"
    },
    "node_id": "MDM6UmVmMTM2NTcxOTAzOnYwLjkuNg=="
  },
  {
    "name": "v0.9.5",
    "zipball_url": "https://api.github.com/repos/moooofly/harbor-go-client/zipball/v0.9.5",
    "tarball_url": "https://api.github.com/repos/moooofly/harbor-go-client/tarball/v0.9.5",
    "commit": {
      "sha": "5dd4268eff2fd75417ec4ddccbea9c8dc539d4ff",
      "url": "https://api.github.com/repos/moooofly/harbor-go-client/commits/5dd4268eff2fd75417ec4ddccbea9c8dc539d4ff"
    },
    "node_id": "MDM6UmVmMTM2NTcxOTAzOnYwLjkuNQ=="
  },
  {
    "name": "v0.9.4",
    "zipball_url": "https://api.github.com/repos/moooofly/harbor-go-client/zipball/v0.9.4",
    "tarball_url": "https://api.github.com/repos/moooofly/harbor-go-client/tarball/v0.9.4",
    "commit": {
      "sha": "eaa6c2b6d2c11d060baaf3ff75d48b1470a71ec0",
      "url": "https://api.github.com/repos/moooofly/harbor-go-client/commits/eaa6c2b6d2c11d060baaf3ff75d48b1470a71ec0"
    },
    "node_id": "MDM6UmVmMTM2NTcxOTAzOnYwLjkuNA=="
  },
  {
    "name": "v0.9.3",
    "zipball_url": "https://api.github.com/repos/moooofly/harbor-go-client/zipball/v0.9.3",
    "tarball_url": "https://api.github.com/repos/moooofly/harbor-go-client/tarball/v0.9.3",
    "commit": {
      "sha": "bcdf9e38f0adafbb9f0dcee5a53347737307adf4",
      "url": "https://api.github.com/repos/moooofly/harbor-go-client/commits/bcdf9e38f0adafbb9f0dcee5a53347737307adf4"
    },
    "node_id": "MDM6UmVmMTM2NTcxOTAzOnYwLjkuMw=="
  },
  {
    "name": "v0.9.2",
    "zipball_url": "https://api.github.com/repos/moooofly/harbor-go-client/zipball/v0.9.2",
    "tarball_url": "https://api.github.com/repos/moooofly/harbor-go-client/tarball/v0.9.2",
    "commit": {
      "sha": "6e72c76af7e1c37b79d8db06fa9dfa78fef20fd9",
      "url": "https://api.github.com/repos/moooofly/harbor-go-client/commits/6e72c76af7e1c37b79d8db06fa9dfa78fef20fd9"
    },
    "node_id": "MDM6UmVmMTM2NTcxOTAzOnYwLjkuMg=="
  },
  {
    "name": "v0.9.1",
    "zipball_url": "https://api.github.com/repos/moooofly/harbor-go-client/zipball/v0.9.1",
    "tarball_url": "https://api.github.com/repos/moooofly/harbor-go-client/tarball/v0.9.1",
    "commit": {
      "sha": "360bc287242c500153f5c7349ca91c07ca53c6bd",
      "url": "https://api.github.com/repos/moooofly/harbor-go-client/commits/360bc287242c500153f5c7349ca91c07ca53c6bd"
    },
    "node_id": "MDM6UmVmMTM2NTcxOTAzOnYwLjkuMQ=="
  },
  {
    "name": "v0.9.0",
    "zipball_url": "https://api.github.com/repos/moooofly/harbor-go-client/zipball/v0.9.0",
    "tarball_url": "https://api.github.com/repos/moooofly/harbor-go-client/tarball/v0.9.0",
    "commit": {
      "sha": "5752a1674d5a1f2262c0e04e8206328e0504d23b",
      "url": "https://api.github.com/repos/moooofly/harbor-go-client/commits/5752a1674d5a1f2262c0e04e8206328e0504d23b"
    },
    "node_id": "MDM6UmVmMTM2NTcxOTAzOnYwLjkuMA=="
  },
  {
    "name": "v0.8.0",
    "zipball_url": "https://api.github.com/repos/moooofly/harbor-go-client/zipball/v0.8.0",
    "tarball_url": "https://api.github.com/repos/moooofly/harbor-go-client/tarball/v0.8.0",
    "commit": {
      "sha": "aecbcfa3570639426ec61382764220b17b80633f",
      "url": "https://api.github.com/repos/moooofly/harbor-go-client/commits/aecbcfa3570639426ec61382764220b17b80633f"
    },
    "node_id": "MDM6UmVmMTM2NTcxOTAzOnYwLjguMA=="
  },
  {
    "name": "v0.7.0",
    "zipball_url": "https://api.github.com/repos/moooofly/harbor-go-client/zipball/v0.7.0",
    "tarball_url": "https://api.github.com/repos/moooofly/harbor-go-client/tarball/v0.7.0",
    "commit": {
      "sha": "baa8fe9b9a252959bbc87fd2b3550da88c4fc6f6",
      "url": "https://api.github.com/repos/moooofly/harbor-go-client/commits/baa8fe9b9a252959bbc87fd2b3550da88c4fc6f6"
    },
    "node_id": "MDM6UmVmMTM2NTcxOTAzOnYwLjcuMA=="
  },
  {
    "name": "v0.6.0",
    "zipball_url": "https://api.github.com/repos/moooofly/harbor-go-client/zipball/v0.6.0",
    "tarball_url": "https://api.github.com/repos/moooofly/harbor-go-client/tarball/v0.6.0",
    "commit": {
      "sha": "c0568b6a0a35228c71203dea04018dd296b14f23",
      "url": "https://api.github.com/repos/moooofly/harbor-go-client/commits/c0568b6a0a35228c71203dea04018dd296b14f23"
    },
    "node_id": "MDM6UmVmMTM2NTcxOTAzOnYwLjYuMA=="
  },
  {
    "name": "v0.5.0",
    "zipball_url": "https://api.github.com/repos/moooofly/harbor-go-client/zipball/v0.5.0",
    "tarball_url": "https://api.github.com/repos/moooofly/harbor-go-client/tarball/v0.5.0",
    "commit": {
      "sha": "d473c2c6a53c93971bbe901bec025ad28e147f6c",
      "url": "https://api.github.com/repos/moooofly/harbor-go-client/commits/d473c2c6a53c93971bbe901bec025ad28e147f6c"
    },
    "node_id": "MDM6UmVmMTM2NTcxOTAzOnYwLjUuMA=="
  },
  {
    "name": "v0.4.0",
    "zipball_url": "https://api.github.com/repos/moooofly/harbor-go-client/zipball/v0.4.0",
    "tarball_url": "https://api.github.com/repos/moooofly/harbor-go-client/tarball/v0.4.0",
    "commit": {
      "sha": "baed9807f4e0c1fc248f9ce3ff464d4b36dcb71e",
      "url": "https://api.github.com/repos/moooofly/harbor-go-client/commits/baed9807f4e0c1fc248f9ce3ff464d4b36dcb71e"
    },
    "node_id": "MDM6UmVmMTM2NTcxOTAzOnYwLjQuMA=="
  },
  {
    "name": "v0.3.0",
    "zipball_url": "https://api.github.com/repos/moooofly/harbor-go-client/zipball/v0.3.0",
    "tarball_url": "https://api.github.com/repos/moooofly/harbor-go-client/tarball/v0.3.0",
    "commit": {
      "sha": "919a7c36dd046bf2fd3afc9aa53ee86e95695c1f",
      "url": "https://api.github.com/repos/moooofly/harbor-go-client/commits/919a7c36dd046bf2fd3afc9aa53ee86e95695c1f"
    },
    "node_id": "MDM6UmVmMTM2NTcxOTAzOnYwLjMuMA=="
  },
  {
    "name": "v0.2.0",
    "zipball_url": "https://api.github.com/repos/moooofly/harbor-go-client/zipball/v0.2.0",
    "tarball_url": "https://api.github.com/repos/moooofly/harbor-go-client/tarball/v0.2.0",
    "commit": {
      "sha": "ad89b5ff5abbd974bafd45b7ee644dc32b2427e6",
      "url": "https://api.github.com/repos/moooofly/harbor-go-client/commits/ad89b5ff5abbd974bafd45b7ee644dc32b2427e6"
    },
    "node_id": "MDM6UmVmMTM2NTcxOTAzOnYwLjIuMA=="
  },
  {
    "name": "v0.1.0",
    "zipball_url": "https://api.github.com/repos/moooofly/harbor-go-client/zipball/v0.1.0",
    "tarball_url": "https://api.github.com/repos/moooofly/harbor-go-client/tarball/v0.1.0",
    "commit": {
      "sha": "07ae5171ca1dd89d17f897cd2d66dcc2c80d490f",
      "url": "https://api.github.com/repos/moooofly/harbor-go-client/commits/07ae5171ca1dd89d17f897cd2d66dcc2c80d490f"
    },
    "node_id": "MDM6UmVmMTM2NTcxOTAzOnYwLjEuMA=="
  }
]
```


## rate limit

调用次数过多时触发 rate limit

```
[#16#root@ubuntu-1604 ~]$github-release info -u moooofly -r harbor-go-client
error: could not fetch tags, expected '200 OK' but received '403 Forbidden' (url: https://api.github.com/repos/moooofly/harbor-go-client/tags?per_page=100)
[#17#root@ubuntu-1604 ~]$
```

超过 rate limit 时 web 上的错误信息

```
{
  "message": "API rate limit exceeded for 47.88.138.42. (But here's the good news: Authenticated requests get a higher rate limit. Check out the documentation for more details.)",
  "documentation_url": "https://developer.github.com/v3/#rate-limiting"
}
```


## 代码研究

```
[#27#root@ubuntu-1604 /go/src/github.com/aktau/github-release]$tree .
.
├── assets.go
├── cmd.go
├── commit.go
├── error.go
├── github
│   ├── debug.go
│   ├── file.go
│   ├── file_test.go
│   └── github.go
├── github-release.go
├── LICENSE
├── Makefile
├── README.md
├── releases.go
├── tags.go
├── term.go
├── util.go
└── version.go

1 directory, 17 files
[#28#root@ubuntu-1604 /go/src/github.com/aktau/github-release]$
```

### tag

```golang
type Tag struct {
	Name       string `json:"name"`
	Commit     Commit `json:"commit"`
	ZipBallUrl string `json:"zipball_url"`
	TarBallUrl string `json:"tarball_url"`
}
```

```
   tag
    ^
    |
  commit
```

### Commit

```
type Commit struct {
	Sha string `json:"sha"`
	Url string `json:"url"`
}
```
  
  
### Release

```golang
type Release struct {
	Url         string     `json:"url"`
	PageUrl     string     `json:"html_url"`
	UploadUrl   string     `json:"upload_url"`
	Id          int        `json:"id"`
	Name        string     `json:"name"`
	Description string     `json:"body"`
	TagName     string     `json:"tag_name"`
	Draft       bool       `json:"draft"`
	Prerelease  bool       `json:"prerelease"`
	Created     *time.Time `json:"created_at"`
	Published   *time.Time `json:"published_at"`
	Assets      []Asset    `json:"assets"`
}
```

```
  release
     ^
     |
  []asset
```

### Asset

```
type Asset struct {
	Url         string    `json:"url"`
	Id          int       `json:"id"`
	Name        string    `json:"name"`
	ContentType string    `json:"content_type"`
	State       string    `json:"state"`
	Size        uint64    `json:"size"`
	Downloads   uint64    `json:"download_count"`
	Created     time.Time `json:"created_at"`
	Published   time.Time `json:"published_at"`
}
```

### 整体

```
        release        tag
           |            |
      +----+---+    +---+---+------+--------+
      |        |    |       |      |        |
   []asset    tag_name     commit  zipUrl  tarUrl
      |                      |
 +----+----+              +--+--+
 |    |    |              |     |
id  name  url            sha    url
```

## 相关链接

- [Rate limiting](https://developer.github.com/v3/#rate-limiting)
- [GitHub Access Token](https://github.com/moooofly/MarkSomethingDown/blob/master/nonsense/GitHub%20Access%20Token.md)
- [Creating a personal access token for the command line](https://help.github.com/articles/creating-a-personal-access-token-for-the-command-line/)

