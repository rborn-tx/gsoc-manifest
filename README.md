This repository contains manifests for building [Torizon OS](https://developer.toradex.com/torizon/) (formerly TorizonCore) and Toradex Reference images.
It was forked from https://git.toradex.com/cgit/toradex-manifest.git/ in order to make adaptations for the Google Summer of Code 2024, particularly to support the following projects:

- [Uptane: Aktualizr Integration With SWUpdate](https://summerofcode.withgoogle.com/programs/2024/projects/qQntgfyx)
- [Building support for an A/B partition scheme-based update for Uptane Client](https://summerofcode.withgoogle.com/programs/2024/projects/8r2exO5W)

# Usage

Basic steps:

1. Make sure prerequisites are fulfilled.
2. Download metadata.
3. Set up build environment.
4. Build OS image.
5. Install OS image onto a device.
6. Update OS of a device when needed.

The steps are detailed in the following items.

## Prerequisites

- Linux host with [prerequisites for building a Yocto/OpenEmbedded image](https://developer.toradex.com/linux-bsp/os-development/build-yocto/build-a-reference-image-with-yocto-projectopenembedded#prerequisites).
- Bash installed and used shell interpreter.

## Download metadata

- Install the Google repo tool:
  ```
  $ mkdir ~/bin
  $ PATH=~/bin:$PATH

  $ curl http://commondatastorage.googleapis.com/git-repo-downloads/repo > ~/bin/repo
  $ chmod a+x ~/bin/repo
  ```

- Make sure your Git user name and e-mail are configured correctly:
  ```
  $ git config --global user.email "you@example.com"
  $ git config --global user.name "Your Name"
  ```

- Check out the metadata for GSoC (inside the `gsoc-2024` directory, as an example):
  ```
  $ mkdir gsoc-2024
  $ cd gsoc-2024
  $ repo init -u https://github.com/rborn-tx/gsoc-manifest -b gsoc-2024 -m gsoc.xml
  $ repo sync -j8
  ```

## Set up build environment

- Enter the build environment:
  ```
  $ cd gsoc-2024

  gsoc-2024:
  $ MACHINE=verdin-imx8mm . setup-environment
  #
  # Current directory should be gsoc-2024/build-torizon
  ```

- Enable the desired updater (RAUC in this example); this is done by editing file `conf/bblayers-updater-benchmarking.conf`.
  ```
  gsoc-2024/build-torizon:
  $ nano conf/bblayers-updater-benchmarking.conf
  #
  # Inside the editor:
  #
  # - Comment out: BENCHMARK_INCLUDED_UPDATER = "none"
  # - Uncomment:   BENCHMARK_INCLUDED_UPDATER = "rauc"
  ```

- **RAUC** specific:

  - Create the required certificates and keys. For this, we use a test script from the rauc repository; the script creates various keys for different purposes; we choose the `autobuilder-1` key and with related cert which is signed by a dev CA which in turn is signed by a root CA. The bundle of CA certs and CRLs are in `dev-only-ca.pem`.
    ```
	gsoc-2024/build-torizon:
	$ mkdir rauc-keys && cd rauc-keys

    gsoc-2024/build-torizon/rauc-keys:
    $ wget 'https://raw.githubusercontent.com/rauc/rauc/v1.11.3/test/openssl-ca-create.sh'
    $ chmod u+x ./openssl-ca-create.sh
    $ ./openssl-ca-create.sh
    $ ln -srf openssl-ca/dev-only-ca.pem ca.cert.pem
    $ ln -srf openssl-ca/dev/autobuilder-1.cert.pem devel-1.cert.pem
    $ ln -srf openssl-ca/dev/private/autobuilder-1.pem devel-1.key.pem
    $ cd ..
    ```

  - Configure the build system to use the desired certificates and keys.
    ```
	gsoc-2024/build-torizon:
    $ nano conf/local.conf
    #
    # Inside the editor:
    #
    # - Set the variables defining certs and keys by adding the following lines to the end of the file (without the hashes):
	#
    #   RAUC_KEYRING_FILE = "${TOPDIR}/rauc-keys/ca.cert.pem"
    #   RAUC_KEY_FILE = "${TOPDIR}/rauc-keys/devel-1.key.pem"
    #   RAUC_CERT_FILE = "${TOPDIR}/rauc-keys/devel-1.cert.pem"
    #
	```

## Build OS image

- Whenever you want to build an OS image:
  ```
  $ cd gsoc-2024

  gsoc-2024:
  $ MACHINE=verdin-imx8mm . setup-environment

  gsoc-2024/build-torizon:
  $ bitbake torizon-minimal
  ```
  
  The resulting image will be produced inside the deployment directory (`deploy/images/<machine>/` (machine in the examples is `verdin-imx8mm`)).

  - **RAUC** specific: image will be named `<image>-<machine>-<date-time>.rauc_tezi.tar`; for example:
	```
    gsoc-2024/build-torizon:
	$ ls deploy/images/verdin-imx8mm/*.rauc_tezi.tar
	deploy/images/verdin-imx8mm/torizon-minimal-verdin-imx8mm-20240603010927.rauc_tezi.tar
	deploy/images/verdin-imx8mm/torizon-minimal-verdin-imx8mm.rauc_tezi.tar
	```
    the second entry above is just a symbolic link to the first one (the actual image as a Toradex Easy Installer tarball).

- **RAUC** specific: whenever you want to build an update bundle:
  ```
  gsoc-2024/build-torizon:
  $ bitbake update-bundle
  ```
  the result will be a file with extension `.raucb` inside the deployment directory (`deploy/images/<machine>/` (machine in the examples is `verdin-imx8mm`)). To learn about other bitbake targets, check the meta-rauc and meta-rauc-community documentation.

## Install OS image

To perform the installation of the OS image use [Toradex Easy Installer](https://developer.toradex.com/easy-installer/toradex-easy-installer/#download-and-install-toradex-easy-installer-image). This tool can load the image from a USB flash drive, an SD card (installed directly on the device) or even via the network, which can be done by the TorizonCore Builder tool, specifically with command [images serve](https://developer.toradex.com/torizon/os-customization/torizoncore-builder-tool-commands-manual/#images-serve).

## Update OS of a device

Once the device is running Torizon OS, it can be updated as needed.

### Updates with RAUC

- On the host/development machine:
  - To determine the host machine IP address to use on the device:
	```
	$ ip a s
	```
  - Serve the `.raucb` file on the network; RAUC requires a server supporting range requests, that's why we're using NGINX here. To serve the file directly from the build output directory:
	```
    gsoc-2024/build-torizon:
    $ cd deploy/images/verdin-imx8mm/
    $ docker run -p 8080:80 -v $(pwd):/usr/share/nginx/html nginx
	```

- On a device:
  - To determine the currently selected "slot":
	```
	$ rauc status
	```
  - To start an update fetching the bundle from the host/development machine:
	```
	$ sudo rauc install http://<host-ip>:8080/update-bundle-verdin-imx8mm.raucb
	# and boot the device on the newly installed OS:
	$ sudo reboot
	```

### Updates with SWUpdate

TODO

# General information about the OS images

- The root filesystem is writable at the moment (this will likely change in the future).
- The latest upstream `aktualizr` (at the time of this writing) is installed into the images by default; the basic configuration allows one to provision the device onto the [Torizon Cloud](https://developer.toradex.com/torizon/torizon-platform/torizon-platform-services-overview/) and contributors are encouraged to create an account into that service.
- The system is set up with an A/B boot system having one partition for each OS deployment; directories `/var` and `/data` both refer to the same separate partition which is common between the A and B systems; this allows state to be shared across updates. In particular both aktualizr and RAUC will keep state information into those directories.


# Useful references

## Toradex

- Toradex Quick Start Guide: https://developer.toradex.com/quickstart (for starting up with a new hardware module or carrier board)
- Toradex Easy Installer: https://developer.toradex.com/easy-installer/toradex-easy-installer
- Verdin iMX8M Mini page: https://developer.toradex.com/hardware/verdin-som-family/modules/verdin-imx8m-mini/

## RAUC

- RAUC documentation: https://rauc.readthedocs.io/en/latest/index.html
- meta-rauc documentation: https://github.com/rauc/meta-rauc
- meta-rauc-community documentation:
  * https://github.com/rborn-tx/meta-rauc-community/tree/rauc-toradex-6.5.0 (our fork of [drewmoseley/meta-rauc-community](https://github.com/drewmoseley/meta-rauc-community) having support for Toradex modules)
  * https://github.com/rauc/meta-rauc-community (upstream, without support for Toradex modules)
