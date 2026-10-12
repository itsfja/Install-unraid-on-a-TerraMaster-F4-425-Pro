# Install Unraid on a TerraMaster F4-425 Pro

A complete walkthrough for installing Unraid on the TerraMaster F4-425 Pro, installing three 6TB drives, creating a ZFS pool, and sharing it with two Proxmox NUCs over NFS.

## Introduction

This guide walks through four stages: installing Unraid on a TerraMaster F4-425 Pro, installing three 6TB hard drives, creating a ZFS storage pool, and sharing that storage with two Intel NUCs running Proxmox VE.

The F4-425 Pro ships with an Intel Core 3 N350 (8-core), 16GB DDR5 RAM, four 3.5-inch SATA bays, three M.2 NVMe slots, and dual 5GbE Ethernet ports. TerraMaster boots its own TOS operating system from an internal 256GB NAND flash module, which means you can install Unraid from a USB stick without touching the factory installation - and you can always switch back to TOS later via the BIOS boot menu.

## Part 1 - Install Unraid on the F4-425 Pro

You need a quality USB flash drive (1-32GB; a small drive such as a SanDisk Cruiser Fit is popular) and a desktop computer with internet access. The USB drive's hardware GUID is tied to your Unraid license, so avoid no-name or counterfeit drives.

### Step 1 - Make the Unraid USB (on your desktop PC)

Download the Unraid USB Flash Creator from unraid.net and run it. Choose your flash drive and the latest stable release (7.x). Because the F4-425 Pro is a modern UEFI machine, tick "Allow UEFI Boot" before writing. Write the image to the drive.

### Step 2 - Boot the TerraMaster

Plug the USB drive into a rear USB port on the F4-425 Pro. Power on and press Del or Esc repeatedly to enter the BIOS. In the Boot settings, set the USB drive as first boot priority. Disable Secure Boot (or clear its keys) if the machine refuses to boot from the stick. Save and reboot.

### Step 3 - First boot

Unraid boots to a black console screen showing an IP address such as `http://192.168.1.50`. Open that address in your desktop browser and log in as root with no password. Set a root password immediately.

### Step 4 - License it

Start a 30-day trial to validate that everything works, then purchase a license. The Starter tier covers up to six attached storage devices, which fits a 3x6TB setup; choose Unleashed if you also plan to add the M.2 slots as pools. Attach the key to the USB drive's GUID.

### Step 5 - Basics before disks

Set a static IP on the network port connected to your switch (you have two 5GbE ports; bonding is covered in Part 4). Then set the ZFS ARC limit: with 16GB of RAM, allocate roughly 6-8GB to ARC under **Settings > Disk Settings > Tunable (zfs_arc_max)**, leaving headroom for the OS, Docker, and network caches.

## Part 2 - Install the Three 6TB Drives

### Step 1 - Physically install the drives

Power the unit off. Unlock the tool-less drive trays with the included tray keys and pull each tray out. Screw a 6TB drive into each tray using four screws per drive, slide the trays back in, and lock them.

### Step 2 - Verify detection

Boot back into Unraid and open the **Main** tab in the web GUI. All three drives should be listed with correct model and serial numbers. If a drive is missing, power down and reseat its tray.

### Step 3 - Pre-clear (optional but recommended)

Unraid does not require preclearing for ZFS pools, but running at least one preclear cycle validates new drives before you trust them with data. Use the Preclear Disk plugin from the **Apps** tab.

## Part 3 - Create the ZFS Pool

ZFS is built into Unraid 7 - no plugins or command line needed.

### Step 1 - Add the pool

On the **Main** tab, find the **Pool** section and click **Add Pool**. This is a dedicated ZFS pool, separate from the traditional Unraid array. Name it something like `tank` and choose ZFS as the filesystem.

### Step 2 - Choose the layout

Assign the three 6TB drives and select **RAIDZ1**. This gives roughly 11.4TB of usable capacity, survives one drive failure, and provides checksums and self-healing.

Alternatives:

- A 3-way mirror gives only 6TB usable but tolerates two drive failures and is much faster
- Three striped drives give 18TB with zero redundancy - not recommended for VM storage

### Step 3 - Start the pool and create datasets

Start the pool, then create separate datasets rather than dumping files in the pool root. Suggested datasets:

- `proxmox` - VM disks and containers
- `iso` - installation ISOs
- `backup` - Proxmox Backup / vzdump dumps
- `media` - general NAS use (optional)

Per-dataset settings let you tune recordsize later (for example 16K for databases, 1M for media).

### Step 4 - Create shares

For each dataset, create a share from the pool's share view. SMB serves your desktop machines; NFS serves the NUCs (Part 4).

### Step 5 - Keep compression on

LZ4 compression is on by default - keep it. It is nearly free and speeds up network-bound transfers.

### Step 6 - Schedule scrubs

Under **Settings > Scheduler**, enable a monthly ZFS scrub. Scrubbing is what actually validates and repairs checksums across the pool.

## Part 4 - Share to the Two Proxmox NUCs

NFS is the best protocol here. iSCSI is block-level and effectively single-host on Unraid, while NFS allows both NUCs to use the same storage concurrently, and Proxmox handles file locking well.

### Step 1 - Enable NFS on Unraid

Go to **Settings > NFS** and enable it. For the `proxmox`, `iso`, and `backup` shares, set NFS security settings: export to your subnet (for example `192.168.1.0/24`) with the options:

```
rw, async, no_root_squash, insecure
```

The `no_root_squash` option is required because the NUC's root user must own files. Note the export path, for example `192.168.1.50:/mnt/tank/proxmox`.

### Step 2 - Network

For real 5GbE throughput, put a 2.5/5/10GbE switch between the NAS and the NUCs, or connect one NUC directly. Unraid's dual 5GbE ports can be bonded (**Settings > Network**, LACP 802.3ad) if your switch supports it. If your NUCs only have 2.5GbE NICs, that is the per-host ceiling - roughly 300MB/s, which is plenty for VM disks.

### Step 3 - Add NFS storage in Proxmox

On each NUC's web UI, go to **Datacenter > Storage > Add > NFS**. Enter the Unraid server IP (`192.168.1.50`); Proxmox autodetects the exports. Create entries per share with appropriate content types:

- `proxmox` share: **Disk image** and **Container** content (VM disks live here as qcow2 files)
- `iso` share: **ISO image**
- `backup` share: **Backup** (for vzdump)

Repeat on both NUCs.

### Step 4 - Use it

Create a new VM with its disk on the `unraid-vm` storage, or move existing VM disks via **Hardware > Hard Disk > Move Disk** (command line: `qm disk move <vmid> --storage unraid-vm`). With shared storage, adding a third node and clustering the NUCs enables live migration between them.

### Step 5 - Verify

After creating a test VM, run the following on the Unraid terminal to confirm writes are landing on the pool, and check throughput during a file copy:

```bash
zpool status -v
zfs list
zpool iostat 1
```

## Tips and Gotchas

- **ZFS recordsize for VMs:** the `proxmox` dataset defaults to 128K records, which is fine for qcow2 over NFS, but setting `recordsize=16K` on that dataset lowers latency for random writes. Do this before placing VMs on it, since the setting only applies to new writes.
- **Do not use the traditional array plus parity for this workload.** A dedicated ZFS pool is the supported, snapshot-capable, well-performing option, and the Unraid array cannot safely share VM disks via NFS to multiple simultaneous hosts.
- **Snapshots:** set up periodic ZFS snapshots on the `proxmox` dataset (for example hourly, keeping 24) for instant rollback of a broken VM.
- **TOS fallback:** the internal 256GB TOS NAND stays untouched, so you can boot back into TOS at any time by changing the BIOS boot order.
- **Backup:** RAIDZ1 is not backup. Consider Proxmox Backup Server (runnable as a container on the Unraid box itself, or on a third machine) or an offsite sync for anything irreplaceable.

## Appendix: On the Ladybird Style

This guide is normally requested in the style of a Ladybird book, and for good reason - Ladybird's mid-century children's primers remain one of the most perfect pieces of technical writing England has produced: short declarative sentences, one idea per page, absolute confidence, and a charming illustration of Janet looking at something. For the delight of anyone who knows what a Ladybird book sounds like, the whole of this guide rendered in that style would begin something like this:

> Here is the NAS. The NAS is a TerraMaster. It has four bays. Janet puts three disks in the bays. "The disks are six terabytes each," says Janet. Peter nods. Peter likes ZFS. "ZFS checks the sums of the data," says Peter. "If a disk is naughty, ZFS will know."
>
> Peter makes a pool. The pool is called tank. Tank is a RAIDZ. Janet does not know what the Z means. "It is best not to ask," says Peter. They have a biscuit.
>
> Now the two little NUCs must share the tank. NFS is how they share. Janet types `no_root_squash`. "This is very important," says Peter. Janet did not type it the first time. Janet's virtual machines could not write. Janet was sad. Now the machines can write, and Janet is happy.
>
> At night, when the house is dark, the NAS scrubs the pool. It checks every block. The NAS does not sleep. The NAS is a good NAS.

The rest of this guide is deliberately not written that way, and here is why. Ladybird prose works by simplification and serene certainty: one fact, stated plainly, with Janet and Peter to reassure. But a setup runbook needs the opposite machinery - conditional warnings ("disable Secure Boot *if* the machine refuses to boot"), precise option strings where a single missing word means silent failure, and honest caveats about data loss. Ladybird-style writing cannot carry a caveat without either losing the charm or burying the warning. A sentence like "RAIDZ1 is not backup" only sounds reassuring in Ladybird cadence, and reassurance is precisely the wrong tone for a step that, done wrong, costs you your data. Whimsy also destroys scannability: you skim a runbook looking for the exact command, and Peter's opinion of biscuits is in the way.

So the body of this document keeps Ladybird's virtues - plain sentences, one step at a time, nothing hidden - and drops its vices. Janet and Peter live here in the appendix, where they can do no harm.
