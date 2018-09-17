# DCO

Ref: https://github.com/apps/dco

> This App enforces the [Developer Certificate of Origin](https://developercertificate.org/) (DCO) on Pull Requests. It requires all commit messages to contain the `Signed-off-by` line with an email address that matches the commit author.

- DCO 作用于 Pull Request
- DCO 要求 commit 中必须包含 `Signed-off-by` 用于匹配 author 的 email ；

> The Developer Certificate of Origin (DCO) is a lightweight way for contributors to certify that they wrote or otherwise have the right to submit the code they are contributing to the project. Here is the full [text of the DCO](https://developercertificate.org/), reformatted for readability:

DCO 是一种确认 contributors 是否具有所有权的轻量级方式；

>> By making a contribution to this project, I certify that:
>> 
>> The contribution was created in whole or in part by me and I have the right to submit it under the open source license indicated in the file; or
>> 
>> The contribution is based upon previous work that, to the best of my knowledge, is covered under an appropriate open source license and I have the right under that license to submit that work with modifications, whether created in whole or in part by me, under the same open source license (unless I am permitted to submit under a different license), as indicated in the file; or
>> 
>> The contribution was provided directly to me by some other person who certified (a), (b) or (c) and I have not modified it.
>> 
>> I understand and agree that this project and the contribution are public and that a record of the contribution (including all personal information I submit with it, including my sign-off) is maintained indefinitely and may be redistributed consistent with this project or the open source license(s) involved.

上面是完整的 DCO 文本内容；

> Contributors **sign-off** that they adhere to these requirements by adding a `Signed-off-by` line to commit messages.

```
This is my commit message

Signed-off-by: Random J Developer <random@developer.example.org>
```

sign-off 的格式；

> `Git` even has a `-s` command line option to append this automatically to your commit message:

```
$ git commit -s -m 'This is my commit message'
```

> Once installed, this integration will set the [status](https://developer.github.com/v3/repos/statuses/) to `failed` if commits in a Pull Request do not contain a valid `Signed-off-by` line.

![](https://cloud.githubusercontent.com/assets/173/24482273/a35dc23e-14b5-11e7-9371-fd241873e2c3.png)


----------


## 实际问题


```
➜  new git:(proposal/a_go_cli_tool_for_harbor) git remote -v
my	git@github.com:moooofly/community.git (fetch)
my	git@github.com:moooofly/community.git (push)
origin	git@github.com:goharbor/community (fetch)
origin	git@github.com:goharbor/community (push)
➜  new git:(proposal/a_go_cli_tool_for_harbor)

➜  new git:(proposal/a_go_cli_tool_for_harbor) git lg1
* c0086b9 - (25 minutes ago) update — moooofly (HEAD -> proposal/a_go_cli_tool_for_harbor, my/proposal/a_go_cli_tool_for_harbor)
* df816e3 - (36 minutes ago) New Proposal: A Go CLI Client for Harbor — moooofly
*   5c9858c - (3 days ago) Merge pull request #5 from goharbor/summarize_meeting_minutes_0913 — Steven Zou (origin/master, origin/HEAD, master)
|\
| * 5c0d16a - (3 days ago) Add meeting minutes for 2018/09/13 meeting — Steven Zou
|/
*   74a7ae7 - (5 days ago) Merge pull request #3 from goharbor/add_proposal_folders — Steven Zou
|\
| * d405cee - (5 days ago) Add the missing expected existing folder 'proposals' — Steven Zou
* |   e857eb1 - (5 days ago) Merge pull request #2 from goharbor/add_proposal_folders — Steven Zou
|\ \
| |/
| * 4ed6afa - (5 days ago) Add proposal folders to keep raisd proposals from community and also update the README to reflect the change — Steven Zou
|/
| * 89faa64 - (10 days ago) Add the very drafted version of governance model docs — Steven Zou (origin/build_governance_model)
|/
* 62f6dc7 - (11 days ago) Add slides link to the meeting minutes doc — Steven Zou
*   06f0cb7 - (11 days ago) Fix conflicts — Steven Zou
|\
| * 2ae792b - (11 days ago) Add meeting minutes of 2018/09/05 — Steven Zou
* | ab4e070 - (11 days ago) Add meeting minutes of 2018/09/05 — Steven Zou
|/
* d1264b0 - (4 weeks ago) Set up repo and add some related materia — Steven Zou
* 7ee0fb2 - (4 weeks ago) Initial commit — Steven Zou

➜  new git:(proposal/a_go_cli_tool_for_harbor) git rebase HEAD~2 --signoff
Current branch proposal/a_go_cli_tool_for_harbor is up to date, rebase forced.
First, rewinding head to replay your work on top of it...
Applying: New Proposal: A Go CLI Client for Harbor
Applying: update
➜  new git:(proposal/a_go_cli_tool_for_harbor)

➜  new git:(proposal/a_go_cli_tool_for_harbor) git push --force my proposal/a_go_cli_tool_for_harbor
Counting objects: 10, done.
Delta compression using up to 4 threads.
Compressing objects: 100% (10/10), done.
Writing objects: 100% (10/10), 1.55 KiB | 1.55 MiB/s, done.
Total 10 (delta 6), reused 0 (delta 0)
remote: Resolving deltas: 100% (6/6), completed with 2 local objects.
To github.com:moooofly/community.git
 + 6f8d98d...c0086b9 proposal/a_go_cli_tool_for_harbor -> proposal/a_go_cli_tool_for_harbor (forced update)
➜  new git:(proposal/a_go_cli_tool_for_harbor)


➜  new git:(proposal/a_go_cli_tool_for_harbor) git log
commit c0086b9278b87bcebad20512ab3a4d3b882a19c1 (HEAD -> proposal/a_go_cli_tool_for_harbor, my/proposal/a_go_cli_tool_for_harbor)
Author: moooofly <centos.sf@gmail.com>
Date:   Mon Sep 17 11:15:59 2018 +0800

    update

    Signed-off-by: moooofly <centos.sf@gmail.com>

commit df816e3e3f197c48a476eb489e62c77c49e87327
Author: moooofly <centos.sf@gmail.com>
Date:   Mon Sep 17 11:05:30 2018 +0800

    New Proposal: A Go CLI Client for Harbor

    Signed-off-by: moooofly <centos.sf@gmail.com>

commit 5c9858c9fa3940f79619551591d11e5819b6339b (origin/master, origin/HEAD, master)
Merge: 74a7ae7 5c0d16a
Author: Steven Zou <loneghost1982@gmail.com>
Date:   Fri Sep 14 13:51:39 2018 +0800

    Merge pull request #5 from goharbor/summarize_meeting_minutes_0913

    Add meeting minutes for 2018/09/13 meeting

commit 5c0d16ad930611e0b9bcbaefefc033d489c57790
Author: Steven Zou <szou@vmware.com>
Date:   Fri Sep 14 13:48:59 2018 +0800

    Add meeting minutes for 2018/09/13 meeting

    Signed-off-by: Steven Zou <szou@vmware.com>
...
```

![DCO failed and resolve method](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/DCO%20failed%20and%20resolve%20method.png)

![DCO success](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/DCO%20success.png)


----------


## signoff

- git-commit

```
➜  ~ man git-commit
...
       -s, --signoff
           Add Signed-off-by line by the committer at the end of the commit log message. The meaning of a signoff depends
           on the project, but it typically certifies that committer has the rights to submit this work under the same
           license and agrees to a Developer Certificate of Origin (see http://developercertificate.org/ for more
           information).
```

- git-rebase

```
➜  ~ man git-rebase
...
       --signoff
           This flag is passed to git am to sign off all the rebased commits (see git-am(1)). Incompatible with the --interactive option.
```

- git-am

```
➜  ~ man git-am
...
       -s, --signoff
           Add a Signed-off-by: line to the commit message, using the committer identity of yourself. See the signoff option in git-commit(1) for more information.
```