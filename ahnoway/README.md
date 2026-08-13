# ahnoway - MSI GL65 Leopard 10SFSK

This directory holds the [isw](https://github.com/YoyPa/isw) fan curve I run on my laptop, whose
hostname is ahnoway, plus the systemd unit that keeps the curve applied after resume.

isw ships no profile for this machine, so I wrote one. The fans on this laptop are reachable only
through the embedded controller, and nothing in the mainline kernel can set a curve on it. That is
the whole reason this directory exists.

## Hardware

| | |
|---|---|
| Model | MSI GL65 Leopard 10SFSK |
| Board | MS-16U7 |
| EC firmware | 16U7EMS1.105 |
| BIOS | E16U7IMS.10B (10/23/2020) |
| CPU | Intel Core i7-10750H |
| GPU | NVIDIA RTX 2070 Super Mobile |
| OS | EndeavourOS, isw 1.10 from the AUR |

Two identifiers matter, and they come from different chips. Check both before you assume this applies
to you.

Board name, from DMI:

```
cat /sys/class/dmi/id/board_name
```

EC firmware version, read from the controller itself at offset 0xa0:

```
sudo dd if=/sys/kernel/debug/ec/ec0/io bs=1 skip=160 count=12 2>/dev/null | tr -d '\0'; echo
```

The second one is the one that counts. isw's profile sections are named after the EC firmware string,
not the BIOS version. DMI will happily tell you your BIOS is `E16U7IMS.10B`, and that string has
nothing to do with which addresses isw writes to. Matching on it would be matching the wrong chip.

That `dd` reads the first 12 bytes, which is the part isw matches on. The full string stored in the
EC is longer and carries a build timestamp after the version: the dump in
[isw issue #146](https://github.com/YoyPa/isw/issues/146) shows it as
`16U7EMS1.1050522202016:22:01`. Read further than 12 bytes and you will see the rest.

If your EC firmware is not `16U7EMS1.105`, treat everything here as unverified.

## Safety

isw writes raw bytes into the embedded controller through the `ec_sys` kernel module with
`write_support=1`. The EC sits below the operating system and owns the fans, battery charging and the
power button directly. Nothing in Linux validates what gets written to it.

The risk is not the numbers in this file. It is the address map. isw decides where to write based on
the profile name you type on the command line, and it never checks what machine you are actually
sitting at. Point it at the wrong profile and you are writing arbitrary bytes into registers that may
not be fan registers.

Before writing anything, dump your own EC and look at what comes back:

```
sudo isw -p 16U7EMS1
```

If the dump reads like a fan curve, six ascending temperatures and seven ascending speeds, then
`MSI_ADDRESS_DEFAULT` is being interpreted correctly on your controller. If it prints zeros, all
`0xff`, or values that make no sense as a curve, stop there. The address map does not fit your
machine.

Two more things worth knowing before you run anything:

isw never verifies its writes. It exits 0 whether or not the bytes landed. A successful exit status
tells you the program ran, not that the curve is in place. Always read the EC back.

There is no fan sensor in hwmon on this laptop, so `sensors` cannot show you fan RPM. `isw -r` is the
only realtime view of what the fans are doing.

## What is here

| File | What it is |
|---|---|
| `16U7EMS1.conf` | The profile section. Append it to `/etc/isw.conf`. |
| `isw-resume.service` | Rewrites the curve after resume, because the EC drops it. |

## The curve

I optimized for sustained cooling and accepted the noise. This is not a quiet profile and it is not
meant to be one.

CPU:

| Temperature | 45 C | 52 C | 58 C | 64 C | 70 C | 76 C |
|---|---|---|---|---|---|---|
| Fan speed | 35% | 50% | 60% | 70% | 85% | 95% |

Above 76 C the fan runs at 100%. There are six thresholds and seven speeds because the first speed
applies below the first threshold.

GPU:

| Temperature | 50 C | 55 C | 60 C | 65 C | 70 C | 75 C |
|---|---|---|---|---|---|---|
| Fan speed | 35% | 45% | 55% | 70% | 80% | 90% |

`fan_mode = 140` is 0x8c, which MSI calls Advanced. That is the mode in which the EC honours a
user-supplied curve rather than its own.

What it measurably changed: light load CPU package temperature dropped from roughly 61 to 65 C down
to roughly 48 to 55 C. Under a full synthetic load the package still peaks at 95 C with the fans at
100%, which is normal for this chassis. Both RAPL limits on this machine, long term and short term,
are set to 200 W against a 45 W rated part, so the cooling system is not the binding constraint.

## Installing it

Install isw from the AUR, then:

```
sudo cp /etc/isw.conf /etc/isw.conf.backup
cat 16U7EMS1.conf | sudo tee -a /etc/isw.conf
sudo isw -w 16U7EMS1
sudo isw -p 16U7EMS1
```

The last command is not optional. It is the only thing that tells you the write landed.

Then enable the boot unit and the resume unit:

```
sudo systemctl enable isw@16U7EMS1.service
sudo cp isw-resume.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable isw-resume.service
```

`isw@.service` ships with isw and applies the curve at boot. `isw-resume.service` is mine, and the
next section explains why it is necessary.

## Why it has to be re-applied

The embedded controller does not keep the curve. It reverts to the factory profile, and nothing in
the system notices, because there is no error and no log entry. The fans simply go back to being
conservative and the machine runs hotter than you think it does.

isw ships `isw@.service` hooked to `sleep.target`, with a two second sleep in it. That does not work
for resume, and the reason is worth understanding.

`sleep.target` is reached on the way down, before the machine actually suspends. So the unit starts,
sleeps two seconds, and races the suspend. On 2026-08-12 I watched it lose: the unit started at
17:00:10 and wrote at 17:00:12, but the machine did not resume until 17:00:37. The write was made
while the machine was on its way into suspend and was discarded.

The fix is to order the unit after `suspend.target` instead. `suspend.target` is ordered
`After=systemd-suspend.service`, and that service does not return until the machine wakes up. So
`suspend.target` is reached only *after* resume, which is exactly when the write needs to happen.

This is a known and long standing systemd behaviour, discussed in
[systemd issue #6364](https://github.com/systemd/systemd/issues/6364).

`isw-resume.service` also runs `isw -p` as `ExecStartPost`, which dumps the EC into the journal after
every resume. Since isw exits 0 whether or not the write landed, that dump is the only durable proof
the curve is actually in place. To read it back:

```
journalctl -u isw-resume.service -b
```

One prerequisite that is easy to miss: a resume hook cannot help you if suspend itself is failing.
On this machine an SSHFS mount was blocking the kernel freezer and aborting suspend entirely, which
looked like a fan problem for a while because the curve kept being wrong. Confirm your machine
actually suspends and resumes before concluding anything about the EC.

## Measured notes

**Changing power source does not reset the EC on this firmware.** isw's own config file says the EC
can be reset "with a reboot or by changing power source". On 16U7EMS1.105 the second half is not
true. I dumped the EC plugged in, unplugged the charger and dumped again, then replugged and dumped a
third time. All three dumps were byte identical, with the transition confirmed each time against
`/sys/class/power_supply/ADP1/online` going 1, 0, 1 and the battery status changing accordingly. I
had planned a udev rule to re-apply the curve on power change and it turned out to be unnecessary.

**`cpu_temp_0 = 45` is dead code on this machine.** The CPU package does not idle below 55 C even
with the aggressive curve applied, so the lowest threshold is never crossed downward and the 35% floor
never engages. I left it in place because it costs nothing and documents the intent.

**The EC hysteresis is wide and asymmetric.** Coming down from load it held 85% at 58 C, which is two
zones above what the curve asks for at that temperature. Steady state generally reads about one zone
higher than commanded. This is EC behaviour, not something the profile controls.

**I have not seen the commanded 35% in practice.** Every `isw -r` reading I have taken has been
higher, which follows from the threshold above it never being crossed. I was not watching
systematically, so treat this as consistent with the dead code note rather than as a separate finding.

## Alternatives considered

I checked whether any of this was necessary before writing a profile. It was.

**Mainline `msi-ec`** exports exactly two sysfs files, both battery charge thresholds. It has no fan
control at all. Adding this firmware to mainline would get you battery thresholds and nothing else.

**Out of tree [BeardOverflow/msi-ec](https://github.com/BeardOverflow/msi-ec)** does support
`16U7EMS1.105`, added in pull request #550 and merged 2026-02-11. It gives you fan mode selection,
Cooler Boost, shift modes, and read only CPU and GPU temperature and fan speed, which is genuinely
more than I have today. It does not do fan curves.

The pull request that would add them,
[#143 "Implement Fan Thermal Curve"](https://github.com/BeardOverflow/msi-ec/pull/143), has been open
since 2024-07-09, with its most recent activity on 2025-07-18. It has not been rejected. Users have
confirmed it working on a GS66-12UHS and a Titan 18HX. What is holding it up is coverage: the author
has only been able to test it on firmware `17L2EMS1.108` and has asked for help porting it to others.
Nobody has tested it on a 16U7. Separately, work is underway to upstream a hwmon compliant fan curve
API for `msi-wmi-platform`, which is a different route to the same feature and, per the note below,
not one this laptop can take.

**`msi-wmi-platform`** cannot bind to this laptop at all. It matches on WMI GUID
`ABBC0F6E-8EA1-11D1-00A0-C90629100000`, which this firmware does not declare. That is a hardware
fact, not a configuration problem.

**`fancontrol` and `lm_sensors`** cannot see the fans. There is no fan channel in hwmon on this
machine, so there is nothing for them to control.

**`thermald`** manages CPU power limits, not fans. It is solving a different problem.

**Upstream isw is effectively unmaintained**, so the missing profile was never going to arrive on its
own. The last commit to master is dated 2020-02-26 and the last release, 1.10, is from 2019-10-25,
which is still what the AUR packages. [Issue #146](https://github.com/YoyPa/isw/issues/146) is an EC
dump for this exact model, a GL65 Leopard 10SFSK-297, opened 2020-12-26 and still unanswered.
[Issue #187](https://github.com/YoyPa/isw/issues/187) posts the same firmware string from a GL65
Leopard 10SEK-465IN, opened 2021-05-05 and also still open. This directory is, in practice, the
answer to both of them.

## Changelog

**2026-08-10, max cooling.** CPU thresholds 45/52/58/64/70/76 with speeds 35/50/60/70/85/95/100. GPU
thresholds 50/55/60/65/70/75 with speeds 35/45/55/70/80/90/100. Roughly 10 C off the light load
package temperature compared to what it replaced.

**2026-02, silence first (superseded).** CPU thresholds 50/56/62/70/75/80 with speeds
0/45/55/65/75/85/95. GPU thresholds 55/60/65/70/75/80 with speeds 0/46/54/62/70/80/90. Written by
hand because isw ships nothing for this board. The GPU half of it turned out to be identical to the
factory curve posted in isw issue #146, both thresholds and speeds, which I only noticed later. The
CPU halves differ.
