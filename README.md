# fan-configs

Linux fan control per machine: an isw profile for the MSI GL65 Leopard 10SFSK, which isw does not
support upstream, plus home lab servers as I get to them.

I built this because isw ships no profile for my laptop and I wanted it to run cooler. The embedded
controller does not keep the curve, so systemd units write it again on boot and after every resume.
I have verified the curve is in place after both.

## Scope

[isw](https://github.com/YoyPa/isw) by YoyPa ships no profile for this hardware: MSI GL65 Leopard
10SFSK, board MS-16U7, EC firmware 16U7EMS1.105.

## Safety

isw writes raw bytes into the embedded controller through `ec_sys` with
`write_support=1`. Nothing in Linux validates those writes, and isw does not
detect your hardware: it writes wherever the profile name you pass it says to.

The values in this repository were measured on one board and one firmware
version. On anything else they are unverified, and a wrong address map means
writing arbitrary bytes into unknown registers.

Read [ahnoway/README.md](ahnoway/README.md) before using any of this.

## Machines

- [ahnoway](ahnoway/) - MSI GL65 Leopard 10SFSK laptop

## Credits and license

The fan control itself is done by [isw](https://github.com/YoyPa/isw) by YoyPa,
which is GPL licensed. This repository does not contain any of isw's code. It
contains a configuration section that isw reads, plus systemd units that call it.

The MIT license in this repository covers those files only: the isw profile
section and the unit files. isw remains under its own license.
