# What BTRFS subvolume mount options should I use

[deleted]

[permalink](http://reddit.com/r/btrfs/comments/shihry/what_btrfs_subvolume_mount_options_should_i_use/)
by *[deleted]* (↑ 9/ ↓ 0)

## Comments

##### &gt; About autodefrag, is it worth it on a SDD and laptop cpu (will there be increased battery drain)?

Not worth it on SSD.

&gt; Or should I use TRIM service instead with discard=async?

discard=async and fstrim do slightly different things. For most users, I'd recommend just sticking with fstrim.

&gt; For compress=zstd, should I go with compress=zstd:1, 2 or default 3? Is there a way to test for the best one on my system? What's the benefits of 1 over default 3?

Smaller number means it spends less effort trying to compress it, so it's a trade-off between CPU time, compression ratio and bandwitdh.

For HDDs and a decent (but modest) CPU, the default levelwill not bottleneck it. On SSDs, especially when you get some NVMe ones, you can bottleneck compressing.

To give a rough estimate, use the zstd utility from userspace:

    zstd -b1 # zstd level one
    zstd -b3 # zstd level three.. and so on.

&gt; For space_cache should I go with v1 or v2? I have a 512 SSD + 16 GB Ram, not sure if v2 is worth it.

V2. It's pretty much always a direct upgrade in reliability and speed.

&gt; is defaults needed? I don't see it being mentioned anymore.

No, it's a placeholder because the fstab format requires _some_ option to exist in order to parse correctly AFAIK.

&gt; Should I disable CoW (nodatacow mount option) on /var/logs and var/cache sub volumes or should I use chattr +C instead for /var? I don't plan to create a /var subvolume but will create one for /var/logs and /var/cache.

As per the documentation (view last point), you cannot disable CoW on a per-subvolume basis, you can only do it via the mount point, otherwise that option may apply to the entire filesystem.

Regarding what you should or should not keep nodatacow, my advice would be: don't overly optimize it, applications already can (and do!) disable nodatacow when they feel like it's needed. For example, look at the systemd journald files.

Furthermore, every nodatacow file you have means many features are disabled, since it's basically an escape hatch, as an example:

* You have NO checksums
* You can't self-heal those files, because it doesn't know what is intact or not.
* You can't compress those files
* etc

&gt; Also does any subvolumes require different mount options or do I use the same mount options for all subvolumes? For example /swap subvolume, do I use the same mount options as the rest or do I need to exclude certain mount options like compress=zstd &amp; autodefrag then add nodatacow (or it's only for the swapfile within the swap subvolume)?

From the wiki:

"Most mount options apply to the whole filesystem and only options in the first mounted subvolume will take effect. This is due to lack of implementation and may change in the future. This means that (for example) you can’t set per-subvolume nodatacow, nodatasum, or compress using mount options. This should eventually be fixed, but it has proved to be difficult to implement correctly within the Linux VFS framework."

"Mount options are processed in order, only the last occurrence of an option takes effect and may disable other options due to constraints (see eg. nodatacow and compress). The output of mount command shows which options have been applied." ⏤ by *Cyber_Faustao* (↑ 9/ ↓ 0)
├─ [deleted] ⏤ by *[deleted]* (↑ 2/ ↓ 0)
├── Yes compression will bottleneck throughput on NVMe drives. Higher compression levels lowers throughput and increases compression time. SATA 6Gbps SSDs will not be limited that much from lower compression levels. Fedora defaults to `compress=zstd:1` so you can copy that if you want compression. ⏤ by *[deleted]* (↑ 3/ ↓ 0)
├─── [deleted] ⏤ by *[deleted]* (↑ 1/ ↓ 0)
├──── `Compress` will skip compression if the beginning of the file does not compress well. `Compress-force` will try to run the compressor on every file (and rely on zstd algorithm to decide which files to skip) and will compress slightly more but at the cost of possibly more CPU usage. 

The choice is up to you (`compress`, `compress-force`, or no compression) and whether or not disk I/O throughput is important for your usage. ⏤ by *[deleted]* (↑ 4/ ↓ 0)
├───── Should be added that zstd's heuristics are "smarter" than of  btrfs', so it's reasonable to use `compress-force=zstd:1` and let zstd decide. ⏤ by *murlakatamenka* (↑ 3/ ↓ 0)
├────── [deleted] ⏤ by *[deleted]* (↑ 2/ ↓ 0)
├─────── `compress-force` will skip incompressible data at the block level based on decisions made by the compression algorithm, `compress` will skip entire files based on some simple heuristics like 'does the first 1MB of this file compress?' You can see where that could break down if a large file starts with incompressible data but contains highly compressible data further in.

BTRFS' skip heuristics were designed years ago when compression algorithms weren't so good at skipping incompressible data efficiently, modern compression algorithms are much better at detecting and skipping incompressible data.

In testing (I don't have data handy, it's been a while since I did this) I've always seen `compress-force` yield better compression results than `compress`, sometimes significantly so. ⏤ by *seaQueue* (↑ 4/ ↓ 0)
├─ &gt;autodefrag

Hi, u/Cyber_Faustao, can you give some details about why autodefrag would be bad with ssds?  We have some servers with large nvme ssds, and their I/O became very slow, and we realized that the issue was fragmentation because after running a defrag the speed shot way up.  After that we enabled autodefrag so that we wouldn't have to manually run defrags regularly.  Is there some disadvantage of this? ⏤ by *gr_eabe* (↑ 2/ ↓ 0)
├── I never said it was bad for SSDs. I said it wasn't worth it.

Most users do not fragment their files enough for it to matter on SSDs, on HDDs I'd recommend it.

The exceptions are stuff like:

* Sqlite DBs from browsers

* VM VHDs.

* journald files

* Torrents

For those, a defrag may be desired, but, yeah, some workloads work better with that option, other times it makes the filesystem have unpredictable performance ⏤ by *Cyber_Faustao* (↑ 2/ ↓ 0)
├─── [deleted] ⏤ by *[deleted]* (↑ 1/ ↓ 0)
├──── &gt; Does TRIM refer to ssd only while autodefrag is meant for HDD?

No. Both options do very different things:

TRIM tells the drive that certain regions (LBAs) are unused and it's free to do some GC or zero them (RZAT).

Also, some HDDs also have TRIM! For example, some SMR drives accept TRIM commands to do the "rearanging chairs on the sinking Titanic" dance of doing garbage collection of its zones.

In any case, TRIM is a device feature, that the OS hints at and may trigger, but the drive itself is responsible for doing its GC and RZAT, etc. The device may also completely ignore TRIM commands from the host, for example, if the TRIM command has too small of an area that the device (SSD/HDD) deems not worth it.

___

Now, autodefrag is purely a BTRFS (read: host computer) thing, the device has no say in this, it simply tries to make your files slightly more contiguous. That being said, latencies on SSDs are 2-3 orders of magnitude smaller than on HDDs, and IOPS are much higher, especially at low queue depths and random access (look at a KDiskMark/CrystaldiskMark report and see the RND4KQ1 value).

In short, even if you have the exact same fragmentation on your SSD, it's not going to make performance degrade too much, or to put it in other words, SSDs don't care too much about fragmentation.

__

&gt; Is it correct that discard=async is basicly improved Continuous TRIM?

Maybe. discard=async will ask the device to TRIM very small areas at a time, which it might choose ot ignore (see previous points). 

But your quote implies it would batch many blocks at a time, not sure if it means:

 A) "batch until we reach a certrain threshold" or..

 B) "batch many unrelated blocks and only call TRIM once instead of many times".

If it's the former, then I'm wrong and it should be perfectly fine for most users (see [1]), if it's the later, my point still stands.

&gt; So for a nvme 512 SSD on a laptop with battery life &amp; cpu usage in concern, is it best to ignore autodefrag and decide between discard=async (Continuous TRIM but not instantaneous trim...) vs ftrim (Periodic TRIM)?

You'd have to try it out. Just beware that autodefrag is busted on kernel 5.16, it will eat your CPU, wait until patches land.

In general, I would ignore the autodefrag on SSD (see the point about SSD fragmentation perf), but just like discard, it depends on your workload, and what's acceptable in terms of latencies, etc.

___ 

[1] - Certain drives, like my own Samsung 850 EVO have a buggy always synchronous  TRIM, even if you ask for discard=async, so it results in IO locking up until the drive finishes doing it's business, because it's a small area, it's pretty fast still, but it still locks up and is very annoying (but otherwise harmless). ⏤ by *Cyber_Faustao* (↑ 3/ ↓ 0)
├───── Just adding note that 5.16.5 fixed auto defrag issue. ⏤ by *Khaneliman* (↑ 1/ ↓ 0)
├─ [deleted] ⏤ by *[deleted]* (↑ 1/ ↓ 0)
├── Setting the +C bit trumps the mount option, so that mount option should have no effect whatsoever on the swap file (if you configured like the wiki says, you know if it's wrong when you can't swapon it without dmesg errors) ⏤ by *Cyber_Faustao* (↑ 2/ ↓ 0)
└────

##### I've used `noatime,compress-force=zstd:1,space_cache=v2,autodefrag` on NVMe for years.

`zstd:1` is appropriate for NVMe drives, it uses the lowest compression level available and yields the best throughput so that fast NVMe isn't waiting on data from the CPU. I'd probably bump the zstd level down to 2 for a SATA SSD and leave it at 3 (or go higher) for HDDs. On faster CPUs or slower drives you can increase the compression level without the compression itself bottlenecking IO to/from the drive.

(edit: The above gets really obvious when you start dealing with systems with excess spare CPU time and very slow storage. You can see a *significant* improvement in storage speeds by increasing compression levels on something like an RPi 4 running a btrfs root off an SD card.)

Its your call if you want to use `compress-force` or not. I prefer to let the compression algorithm's skipping logic make the call to skip blocks if needed instead of btrfs' heuristics, you'll get better compression results this way and I don't see a significant slowdown in practice.

I prefer periodic trim. `discard=async` yields unpredictable storage performance at unpredictable times so I use the `fstrim.timer` supplied by my distro.

`autodefrag` might not be worth it for some people, I leave it on to reduce fragmentation over time and extend the time between manual defrag passes. This means snapshots stay smaller longer without fragmentation becoming *too* bad. I do a lot of small file rewrites, your use case may be different.

For a swapfile you'll want to follow setup instructions very carefully: https://wiki.archlinux.org/title/btrfs#Swap_file, there are a whole laundry list of gotchas.

`space_cache=v2` is a straight upgrade.

Only the first mount entry for the filesystem in fstab needs most options, for the first entry (root fs) I use

    rw,noatime,compress-force=zstd:1,autodefrag,subvol=/Arch/@root

Later mount points use something like

    rw,noatime,subvol=/@home

Note: `space_cache=v2` only needs to be set (and converted to) once, it's persistent on disk after that. It's the default on new filesystems starting in kernel 5.15 or 5.16 ⏤ by *seaQueue* (↑ 5/ ↓ 0)
├─ [deleted] ⏤ by *[deleted]* (↑ 1/ ↓ 0)
├── No, there's no full file compression then comparison. Btrfs `compress` implements some simple heuristics by sampling the data to decide whether to pass it to the compression algo or not. Modern compression algorithms (zstd, lz4, etc) have "skip" functionality baked into their compression pass that serves the same purpose. Using `compress-force` skips btrfs' heuristics and sends all data through the compression pipeline where zstd skips incompressible chunks as needed.

It's more a question of "do I trust btrfs' heuristics here or do I trust the algorithm?" I trust zstd to make better decisions here, and in practice I've seen better compression results when I've tested both options. I don't think you'll see a significant difference in power consumption by picking either one but you could always run some tests and check for yourself. ⏤ by *seaQueue* (↑ 4/ ↓ 0)
├─── Even when using `compress-force` BTRFS will still check whether the compression was effective afterwards. It will pass all the data to the compressor, but if the resulting compressed data is the same size or larger, it will write the data do the disk uncompressed. You can use tools like `compsize` to inspect the compressions on your filesystem. Here's my root subvolume for example:

    ❯ sudo compsize -x /                         
    Processed 366867 files, 251260 regular extents (258012 refs), 186764 inline.
    Type       Perc     Disk Usage   Uncompressed Referenced  
    TOTAL       42%      6.5G          15G          16G       
    none       100%      1.7G         1.7G         1.8G       
    lzo         58%      2.8M         4.9M         4.9M       
    zstd        35%      4.8G          13G          14G

So 1.7G of my data wasn't worth compressing, so it's stored uncompressed. 4.6M of my data was initially compressed with lzo, when my system was installed, where the installer defaulted to plain `compress`, and 13G of my data has been compressed using zstd, which I switched to immediately after installing. The result is that overall compression has reduced the actual disk space required to only 42% of it's uncompressed size. ⏤ by *FrederikNS* (↑ 3/ ↓ 0)
└────

##### On SSD on Manjaro, I use:

    rw,noatime,compress-force=zstd:1,ssd,space_cache

and a trim service runs once a week. ⏤ by *billdietrich1* (↑ 2/ ↓ 0)
├─ You should upgrade to space\_cache v2 as it is the default for all new Btrfs filesystems now. ⏤ by *[deleted]* (↑ 2/ ↓ 0)
├── I think Manjaro created it that way when I installed, 4 or so months ago. ⏤ by *billdietrich1* (↑ 1/ ↓ 0)
├─── Yes, it probably did. Change to btrfs-progs to make space\_cache=v2 default was merged back in December: [https://github.com/lansuse/btrfs-progs/commit/26852f45c4220f9d7978fd76d2a958668550f7c6](https://github.com/lansuse/btrfs-progs/commit/26852f45c4220f9d7978fd76d2a958668550f7c6)

Anyways, it is recommended that you boot into a live usb environment and switch over to space\_cache=v2 by mounting your Btrfs partition once with `-o clear_cache,space_cache=v2`. After that, it should stay on v2 without these mount options. ⏤ by *[deleted]* (↑ 2/ ↓ 0)
├──── I'll just wait until the next time I distro-hop, see if next distro uses v2 by default. ⏤ by *billdietrich1* (↑ 1/ ↓ 0)
└────

##### Default zstd level of 3 is very good, don't go below unless you have a PCIe 4.0 Nvme drive or multiple SSDs in RAID0 and want to utilize their full write speed.
Consider using compress-force, zstd is pretty good at quickly estimating compressibility of content itself.

space_cache=v2 only seems to be recommended for big volumes, over 1 TB, where it can lead to a slight performance increase. And when using Btrfs RAID56 with metadata in RAID1, for better data safety. 

The order of options does not matter.

Do not use autodefrag for SSDs.

The ssd option is redundant when used on SSDs.

It is not possible to set the nodatacow mount option per subvolume, it won't make a difference.

Mount options are passed through to further subvolumes on the same filesystem, unless a contradictory mount option is set. But not all mount options can be set per subvolume, only a few I think. ⏤ by *damster05* (↑ 2/ ↓ 0)
├─ [deleted] ⏤ by *[deleted]* (↑ 2/ ↓ 0)
├── Looks like I underestimated the write speed penalty of zstd:3, it is probably quite different depending on content.  


I don't understand, didn't zstd:1 perform better for you? It definitely should, unless  you are bottlenecked by your SSD's write speed, which is when better compression leads to higher throughput.   


Switching to zstd:1 is completely fine, it won't change any of the already written data, just future writes.  


As I said, it is not possible to disable CoW on specific subvolumes with `nodatacow`, but I would recommend running `chattr +C` on it, or `chattr -R +C` for the entire directory recursively. ⏤ by *damster05* (↑ 2/ ↓ 0)
├─── [deleted] ⏤ by *[deleted]* (↑ 2/ ↓ 0)
├──── Yes, then use that, unless you care more about compression than write speeds or additional cpu load.

Read speeds are not affected by any significant amount by zstd compression level. ⏤ by *damster05* (↑ 2/ ↓ 0)
└────

##### don't ever defrag an SDD. they are fragmented by nature and you will shorten its life drastically. ⏤ by *[deleted]* (↑ 1/ ↓ 0)
├─ As a rule, don't defrag an SSD because there's no physical seek time and it doesn't really impact performance, so you're just wasting write cycles for no benefit - not sure what you mean by fragmented by nature.

But it isn't quite as apocalyptic as you make out, modern SSDs are pretty robust. ⏤ by *CorrosiveTruths* (↑ 2/ ↓ 0)
├─ [deleted] ⏤ by *[deleted]* (↑ 1/ ↓ 0)
├── Most mount options default to off, it isn't because they're unsafe to use, just that mount is very conservative. Some defaults don't interact very well with CoW, [relatime for example](https://btrfs.wiki.kernel.org/index.php/FAQ#Why_I_experience_poor_performance_during_file_access_on_filesystem.3F) especially with oodles of read-write snapshots like with Timeshift.

autodefrag has a performance penalty to prevent future performance penalties. The [documentation gives examples of possible fragmentation issues](https://btrfs.wiki.kernel.org/index.php/Gotchas#Fragmentation) on both HDD and SSD (on SSD it would be cpu spikes rather than thrashing of course) what can cause them and how to deal with them (including autodefrag). ⏤ by *CorrosiveTruths* (↑ 2/ ↓ 0)
├─── [deleted] ⏤ by *[deleted]* (↑ 1/ ↓ 0)
├──── I wouldn't rule out a negligible impact, but no, it won't shorten the life of your drive.

It's a reasonable comment that you might never see an issue and even if you do, you could deal with that then. I'm not overly concerned with performance tuning and set it so I'd not have to worry about that sort of thing down the road. ⏤ by *CorrosiveTruths* (↑ 3/ ↓ 0)
├───── [deleted] ⏤ by *[deleted]* (↑ 1/ ↓ 0)
├────── That makes sense, but bear in mind that in terms of battery life, compression will have a huge impact in comparison to autodefrag. ⏤ by *CorrosiveTruths* (↑ 2/ ↓ 0)
├── I stand corrected on btrfs autodefrag and SSDs. I've been using btrfs for everything for 4 years. I've never experienced the kind of nightmares many report re data loss, but I kinda attribute that to keeping my mount options simple. I wouldn't use the option. but I like to play it safe. that's the best advice I can give. ⏤ by *[deleted]* (↑ 1/ ↓ 0)
└────

