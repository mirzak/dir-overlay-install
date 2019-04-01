### Specification

|||
| --- | --- |
|Module name| dir-overlay-install |
|Supports rollback|yes|
|Requires restart|no|
|Artifact generation script|yes|
|Full system updater|no|
|Source code|[Update Module](https://github.com/mirzak/dir-overlay-install/blob/master/modules/dir-overlay-install), [Artifact Generator](https://github.com/mirzak/dir-overlay-install/blob/master/modules-artifact-gen/dir-overlay-install-artifact-gen)|

#### Prerequisites

Generating Mender Update modules requires you to have the  Mender Artifact tool installed on your system. Instructions on how to install can be found [here](https://docs.mender.io/artifacts/modifying-a-mender-artifact#compiling-mender-artifact).

#### Functional description

The Directory Overlay Install Update Module installs a user defined file tree structure into a given destination directory in the target.

Before the deploy into the destination folder on the device, the Update Module will take a backup copy of the current contents, allowing for restore of it using the rollback mechanism of the Mender client if something goes wrong. The Update Module will also delete the current installed content that was previously installed using the same module, this means that each deployment is self contained and there is no residues left on the system from the previous deployment.

### Device requirements

The `dir-overlay-install` module will store state information in `/var/lib/mender`, and that is:

- Manifest files of current and previous installation
    - list of files
- Backup of current installed files based on previous manifest, which is used in case of roll-back

Sample directory layout:

```bash
$ ls /var/lib/mender/dir-overlay-install/
backup.tar     manifest       manifest.prev
```

The `dir-overlay-install` directory is created by the module.

### Installation on device

This Update Module is out-of-the-box for Mender client 2.0 and later versions. If you need to install from sources, you can download the latest version directly to your device with:

    mkdir -p /usr/share/mender/modules/v3 && wget -P /usr/share/mender/modules/v3 https://raw.githubusercontent.com/mirzak/dir-overlay-install/master/modules/dir-overlay-install && chmod +x /usr/share/mender/modules/v3/dir-overlay-install

### Artifact creation

For convenience, an Artifact generator tool `dir-overlay-install-artifact-gen` is provided along the module. This tool will generate Mender Artifacts in the same format that the Update Module expects them.

To download `dir-overlay-install-artifact-gen`, run the following:

```bash
wget https://raw.githubusercontent.com/mirzak/dir-overlay-insall/master/modules-artifact-gen/dir-overlay-install-artifact-gen
```
Make it executable by running,

```bash
chmod +x dir-overlay-install-artifact-gen
```

Lets create example root filesystem content to deploy,

```bash
mkdir -p rootfs_overlay/etc rootfs_overlay/usr/bin rootfs_overlay/usr/share/app
touch rootfs_overlay/etc/app.conf
touch rootfs_overlay/usr/bin/app
touch rootfs_overlay/usr/share/app/data.txt
```

And the result should look like this:

```bash
$ tree rootfs_overlay/
rootfs_overlay/
├── etc
│   └── app.conf
└── usr
    ├── bin
    │   └── app
    └── share
        └── app
            └── data.txt

5 directories, 3 files
```

You can now generate your Artifact using the following command:

```bash
ARTIFACT_NAME="my-update-1.0"
DEVICE_TYPE="my-device-type"
OUTPUT_PATH="my-update-1.0.mender"
DEST_DIR="/"
OVERLAY_TREE="rootfs_overlay"
./dir-overlay-install-artifact-gen -n ${ARTIFACT_NAME} -t ${DEVICE_TYPE} -d ${DEST_DIR} -o ${OUTPUT_PATH} ${OVERLAY_TREE}
```

- `ARTIFACT_NAME` - The name of the Mender Artifact
- `DEVICE_TYPE` - The compatible device type of this Mender Artifact
- `OUTPUT_PATH` - The path where to place the output Mender Artifact. This should always have a `.mender` suffix
- `DEST_DIR` - The path on target device where content of `OVERLAY_TREE` will be installed.
- `OVERLAY_TREE` - The path a folder containing the contents to be installed on top of the `DEST_DIR` on the device in the update.

You can either deploy this Artifact in managed mode with the Mender server as described in [Deploy to physical devices](https://docs.mender.io/getting-started/deploy-to-physical-devices#upload-the-artifact-to-the-server) or by using the Mender client only in [Standalone deployments](https://docs.mender.io/architecture/standalone-deployments?target=_blank).

#### Artifact technical details

The Mender Artifact used by this Update Module has three payload files:

- `update.tar` - a tarball containing all the files to be deployed
- `dest_dir` -  a regular file with the `DEST_DIR` in plain text
- `manifest` - a regular file with a list of files installed in plain text

The Mender Artifact contents will look like:

```bash
Artifact my-update-1.0.mender generated successfully:
Mender artifact:
  Name: my-update-1.0
  Format: mender
  Version: 3
  Signature: no signature
  Compatible devices: '[my-device-type]'
  Provides group:
  Depends on one of artifact(s): []
  Depends on one of group(s): []
  State scripts:

Updates:
    0:
    Type:   dir-overlay-install
    Provides: Nothing
    Depends: Nothing
    Metadata: Nothing
    Files:
      name:     update.tar
      size:     10240
      modified: 2019-04-01 12:01:41 +0000 UTC
      checksum: fabfd7f6739d35130dd05dbab42ff7fe8baa76fea22f134b254671bd52604807
    Files:
      name:     dest_dir
      size:     2
      modified: 2019-04-01 12:01:41 +0000 UTC
      checksum: f465c3739385890c221dff1a05e578c6cae0d0430e46996d319db7439f884336
    Files:
      name:     manifest
      size:     48
      modified: 2019-04-01 12:01:41 +0000 UTC
      checksum: fc582174946ec48e2782ed9357e9d4389cc0e2a3bfbc261b418d4fd5a326d472
```
