# PPA 相关

## 使用场景示例

> If running Ubuntu < 15.04, you’ll need to install from a different PPA. We recommend [chris-lea/redis-server](https://launchpad.net/~chris-lea/+archive/ubuntu/redis-server)

在某些发行版中，由于内置的软件包版本比较低，因此需要自行添加“源”，以安装期望的版本；

针对上述场景，细节如下：

- Adding this PPA to your system

You can update your system with **unsupported packages** from this **untrusted PPA** by adding `ppa:chris-lea/redis-server` to your system's Software Sources. 

> 相当于自动安装

```
sudo add-apt-repository ppa:chris-lea/redis-server
sudo apt-get update
```

- Technical details about this PPA

> 相当于手动安装

This PPA can be added to your system **manually by copying** the lines below and adding them to your system's software sources.

Display `sources.list` entries for:

[ Zesty (17.04) ]

```
deb http://ppa.launchpad.net/chris-lea/redis-server/ubuntu zesty main 
deb-src http://ppa.launchpad.net/chris-lea/redis-server/ubuntu zesty main 
```

[ Xenial (16.04) ]

```
deb http://ppa.launchpad.net/chris-lea/redis-server/ubuntu xenial main 
deb-src http://ppa.launchpad.net/chris-lea/redis-server/ubuntu xenial main 
```

[ Vivid (15.04) ]

```
deb http://ppa.launchpad.net/chris-lea/redis-server/ubuntu vivid main 
deb-src http://ppa.launchpad.net/chris-lea/redis-server/ubuntu vivid main 
```


Signing key:

```
1024R/136221EE520DDFAF0A905689B9316A7BC7917B12 (What is this?)
```

Fingerprint:

```
136221EE520DDFAF0A905689B9316A7BC7917B12
```


----------

## How do I use software from a PPA?

To start installing and using software from a **Personal Package Archive**, you first need to tell Ubuntu where to find the PPA.

> **Important**: The contents of Personal Package Archives are **not** checked or monitored. You install software from them **at your own risk**.

If you're using the most recent version of Ubuntu (or any version from Ubuntu 9.10 onwards), you can add a PPA to your system with a single line in your terminal.

**Step 1**: On the PPA's overview page, look for the heading that reads `Adding this PPA to your system`. Make a note of the PPA's location, which looks like:

```
ppa:gwibber-daily/ppa
```

**Step 2**: Open a terminal and enter:

```
sudo add-apt-repository ppa:user/ppa-name
```

Replace `ppa:user/ppa-name` with the PPA's location that you noted above.

Your system will now **fetch the PPA's key**. This enables your Ubuntu system to verify that the packages in the PPA have not been interfered with since they were built.

**Step 3**: Now, as a one-off, you should tell your system to **pull down the latest list of software** from each archive it knows about, including the PPA you just added:

```
sudo apt-get update
```

Now you're ready to start installing software from the PPA!

----------


## [Packaging/PPA](https://help.launchpad.net/Packaging/PPA)

### Overview

Using a **Personal Package Archive (PPA)**, you can **distribute** software and updates directly to Ubuntu users. **Create** your source package, **upload** it and Launchpad will **build** binaries and then **host** them in your own apt repository.

That means Ubuntu users can install your packages in just the same way they install standard Ubuntu packages and they'll automatically receive updates as and when you make them.

Every individual and team in Launchpad can have one or more PPAs, each with its own URL.

Packages you publish in your PPA will remain there until you remove them, they're superseded by another package that you upload or the version of Ubuntu against which they're built becomes obsolete.

> **Note**: CommercialHosting allow you to have private PPAs.

#### Size and transfer limits

Each PPA gets 2 GiB of disk space. If you need more space for a particular PPA, ask us.

While we don't enforce a strict limit on data transfer, we will get in touch with you if your data transfer looks unusually high.

#### Supported architectures

When Launchpad builds a source package in a PPA, by default it creates binaries for:

- x86
- AMD64

You may also request builds for **arm64**, **armhf**, and/or **ppc64el**. Use the "Change details" page for the PPA to enable the architectures you want.

Changing the set of architectures for which a PPA builds does not create new builds for source packages that are already published in that PPA; it only affects which builds will be created for new uploads. If you need to create builds for newly-enabled architectures without reuploading, go to "View package details" and then "Copy packages", select all the packages for which you want to create builds, select "This PPA", "The same series", and "Copy existing binaries", and submit the form using the "Copy Packages" button.

We use **OpenStack clouds** for security during the build process, ensuring that each build has a clean build environment and different developers cannot affect one another's builds accidentally. These clouds do not yet have support for the **powerpc** and **s390x** architectures; when they do, it will also be possible to request those architectures in PPAs.

#### Supported series

When building a source package you can specify one of the supported series in your changelog file which are listed at the Launchpad PPA page.

If you specify a different series the build will fail.

### Activating a PPA

Before you can start using a PPA, whether it's your own or it belongs to a team, you need to **activate** it on your profile page or the team's overview page. If you already have one or more PPAs, this is also where you'll be able to create additional archives.

#### Your PPA's key

Launchpad **generates** a unique key for each PPA and uses it to **sign** any packages built in that PPA.

This means that people downloading/installing packages from your PPA can verify their source. After you've activated your PPA, uploading its first package causes Launchpad to start generating your key, which can take up to a couple of hours to complete.

Your key, and instructions for adding it to Ubuntu, are shown on the PPA's overview page.

### Deleting a PPA

When you no longer need your PPA, you can delete it. This deletes all of the PPA's packages, and removes the repository from ppa.launchpad.net. You'll have to wait up to an hour before you can recreate a PPA with the same name.

### Next steps

You can familiarise yourself with how PPAs work by [installing a package from an existing PPA](https://help.launchpad.net/Packaging/PPA/InstallingSoftware). You can also jump straight into [uploading your source packages](https://help.launchpad.net/Packaging/PPA/Uploading).