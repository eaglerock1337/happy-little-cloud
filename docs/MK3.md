# Happy Little Cloud Mark 3

## Stage 1 - Let's Try Again

- Stage goals:
  - Get a test set of Raspberry Pi 5's running
  - Figure out a permanent per-pi storage solution
  - Deploy a proper solution (e.g. Rook) for managing storage
  - Replace the previous `hlc-401`-based storage
  - Prove out Prometheus and Grafana tracking

### Stage 1.1 - New Storage Solution

- Raspberry Pis support NVMe!
  - [PineBerry HatDrive!](https://www.amazon.com/Pineboards-HatDrive-NVMe-2230-Raspberry/dp/B0CTYGDW9T)
    - Supports 2230 and 2242 form factors
    - PCIe Gen 3.0 Support
  - [Corsair MP600 Micro](https://www.amazon.com/Corsair-MP600-Micro-NVMe-PCIe/dp/B0CP68JNT1?crid=2R4DJHT9HFR3J&dib=eyJ2IjoiMSJ9.XbsUxNzPbkwgMB1DsIccQJaZokBJtm26UeH299xG6swdgJmfK53H5mwjfGaDL6TwNNbk_Ec6L3J-4-dnl5OtPCK614D05dpowzESSANvXGqcc5Qm8G9OVRI3IPwQ5l2Dkd5tp4wNAQ3sqeo0-cRUYvP2rMI_8R9ak8zNi7yoV0OvKHKlnJhfctMlthw4FPE25__A7lH-caM-IWlAfMRRC8dx5zfRlKxJIhWMG97JdPtsAteBgGD1BafVgd2kugSxx-QLqKGXKAOvJirepNrxfKY7z5wTjnlvXPUxms7uTpw.ewV-aY1mYZFPUp9sV11pW9XcSFrGUsOf18IpobbgjfA&dib_tag=se&keywords=corsair+nvme+1tb&qid=1744594222&s=electronics&sprefix=corsair+nvme+1tb%2Celectronics%2C97&sr=1-4)
    - PCIe Gen 4.0 Support
    - 1 TB x 8 for 8TB of raw storage
    - Can work with 1, 2, or 3 replicas on a PVC basis
  - Storage will be consumed as a mounted filesystem
  - Options for Block storage:
    - [Rook](https://rook.io/docs/rook/latest-release/Getting-Started/intro/)
    - [Longhorn](https://longhorn.io/docs/1.8.1/deploy/install/)
    - Longhorn is [supported by k3s](https://docs.k3s.io/storage#setting-up-longhorn)

### Stage 1.2 - Raspberry Pi 5 Baremetal Setup

- Use same images as last time
  - No Raspberry Pi 5 images available [on unofficial site](https://raspi.debian.net/tested-images/)
  - The Raspbery Pi 4 images should be safe to use [upon code review](https://salsa.debian.org/raspi-team/image-specs/-/blob/master/generate-recipe.py?ref_type=heads)
  - Downloading latest snapshot and running SD card setup:
    - `sysconfig.txt` - set hostname and authorized key
    - `cmdline.txt` - append `cgroup_enable=memory cgroup_memory=1`

- Switch to Armbian
  - Closest to pure Debian
  - Has a lot of great out of the box setup stuff
  - Can install with the Raspberry Pi Installer
  - Get at [the RPi download page](https://www.raspberrypi.com/software/)
  - Flash Armbian for rpi5
  - OS Customization Settings
    - Most might get overwritten by autoconf
    - Set hostname (e.g. hlc-501)
    - Set username and password - eaglerock/pass
    - Configure Wireless LAN - NO
    - Locale Settings:
      - Time zone: America/New_York
      - Keyboard Layout: us
    - Enable SSH
  - Mount disk
    - `vi /root/.not_logged_in_yet`
    - Settings to use:

```text
PRESET_NET_CHANGE_DEFAULTS="1"
PRESET_NET_ETHERNET_ENABLED="1"
PRESET_NET_WIFI_ENABLED="0"
PRESET_NET_USE_STATIC="1"
PRESET_NET_STATIC_IP="10.23.50.<ip address>"
PRESET_NET_STATIC_GATEWAY="255.255.255.0"
PRESET_NET_STATIC_DNS="10.23.1.120 10.23.1.1"
SET_LANG_BASED_ON_LOCATION="n"
PRESET_LOCALE="en_US.UTF-8"
PRESET_TIMEZONE="America/New_York"
PRESET_USER_NAME"eaglerock"
PRESET_USER_PASSWORD="<password>"
PRESET_DEFAULT_REALNAME="Peter Marks"
PRESET_USER_SHELL="bash"
```

- Problems with install prevent moving forward
  - These configs seem good, but devices won't power up
    - Raspberry Pi 5 power requirements are 25W, 5V, 5A
    - Need individual power supplies
  - New power infrastructure:
    - Decommission 2 Anker USB hubs
    - Reuse 1-to-2 Type-A power adapter splitter
    - Use 2 compact 2x3 power strip ([Amazon](https://www.amazon.com/dp/B0BW3824JN?ref=ppx_yo2ov_dt_b_fed_asin_title))
    - Use 8x Raspberry Pi 5 USB-C Power Adapter ([Amazon](https://www.amazon.com/dp/B0CQLNP9DZ?ref=ppx_yo2ov_dt_b_fed_asin_title))
    - This will interfere with the power strip in Rack slot 9
    - 90-Degree USB-C Male-to-Female Adapter fixes this ([Amazon](https://www.amazon.com/dp/B0CQH48YFQ?ref=ppx_yo2ov_dt_b_fed_asin_title))

### 1.3 Interlude - Fix the Existing Cluster

- Current issues:
  - I'm having issues with authenticating to the cluster from my workstation
    - It keeps timing out and will work only sparingly
    - I need to manage the cluster by logging in to one of the control-plane nodes
  - ArgoCD does not have full information
    - Has an issue with loading data and cache
    - Argo works fine, but the UI doesn't work correctly, only displaying basic information
- Fixes:
  - Authentication and working on the command-line with `kubectl`
    - Trying to use Helm on `hlc-402` failed, so I'm resorting to fixing my desktop access again
    - I follow my bootstrap steps, including copying the kubeconfig from the control plane node
    - Works perfectly afterwards...looks like each control-plane might have its own kubeconfig?
    - Either way, I now know how to regain access to my cluster and don't need to fight with logging in and waiting for it to work
    - Could also be due to `hlc-402` being an upgraded version of k8s, so that might be the reason for a new kubeconfig
  - ArgoCD
    - Now that I have my `kubectl` access fixed, I try a Helm upgrade of argocd...still fails
    - Do some googling, someone recommends restarting the `argocd-application-controller` `statefulset`
    - This fails
    - However, I _do_ notice the `argocd-redis-ha-server` statefulset is unhealthy
    - Fixing this fixes everything...it was literally a cache issue the whole damn time
    - Now I know for next time, while ArgoCD is stateless, it has throwaway state in redis cache

## Future Setup

- Upgrade to latest k8s supported by `k3s`
- Upgrade ArgoCD to latest version as well
- Using NVMe as root partition
  - [Armbian Forum find for nanopc that should work](https://forum.armbian.com/topic/13617-nanopc-t4-boot-from-a-sdcard-use-the-nvme-as-the-root-partition-quick-and-very-dirty/)