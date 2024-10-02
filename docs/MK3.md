# Happy Little Cloud Mark 3

## Stage 1 - Let's Try Again

- Stage goals:
  - Get a test set of Raspberry Pi 5's running
  - Figure out a permanent per-pi storage solution
  - Deploy a proper solution (e.g. Rook) for managing storage
  - Replace the previous `hlc-401`-based storage
  - Prove out Prometheus and Grafana tracking

### Stage 1.1 - New Storage Solution

TBD

### Stage 1.2 - Raspberry Pi 5 Baremetal Setup

- Use same images as last time
  - No Raspberry Pi 5 images available [on unofficial site](https://raspi.debian.net/tested-images/)
  - The Raspbery Pi 4 images should be safe to use [upon code review](https://salsa.debian.org/raspi-team/image-specs/-/blob/master/generate-recipe.py?ref_type=heads)
  - Downloading latest snapshot and running SD card setup:
    - `sysconfig.txt` - set hostname and authorized key
    - `cmdline.txt` - append `cgroup_enable=memory cgroup_memory=1`
  - blah
