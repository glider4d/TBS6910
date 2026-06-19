# TBS6910 - driver setup and signal test

## 1. Install build dependencies

```bash
apt install -y git build-essential libncurses-dev libssl-dev libelf-dev bison flex dwarves zstd bc
```

## 2. Get and build the driver

```bash
mkdir -p ~/repo/tbsdriver && cd ~/repo/tbsdriver
git clone https://github.com/tbsdtv/media_build.git
git clone --depth=1 https://github.com/tbsdtv/linux_media.git -b latest ./media

cd media_build
make dir DIR=../media
make allyesconfig
make -j4
```

Takes 10-20 min. Look for these two lines near the end:

```
LD [M]  netup-unidvb.ko
LD [M]  tbsecp3.ko
```

`Skipping BTF generation` warnings are harmless, ignore them. Anything under `error:` is not.

## 3. Install

```bash
sudo make install
sudo depmod -a
sudo modprobe -r tbsecp3 2>/dev/null
sudo modprobe tbsecp3
```

## 4. Check the card is detected

```bash
dmesg | grep -i tbs
ls /dev/dvb
```

Expected:

```
TBSECP3 driver 0000:04:00.0: TurboSight TBS 6910 DVB-S/S2 + 2xCI
adapter0  adapter1
```

If a CAM module is in the CI slot, you'll also see:

```
dvb_ca_en50221: dvb_ca adapter 0: DVB CAM detected and initialised successfully
```

## 5. Check tuner capabilities

```bash
dvb-fe-tool -a 0
```

Should list `DVBS` / `DVBS2` and a frequency range of 950 MHz - 2.15 GHz.

The "operation not permitted" message at the end is just the tool trying to power off the LNB on exit - safe to ignore.

## 6. Transponder config (Intersputnik)

`intersputnik.conf`:

```ini
[INTERSPUTNIK]
DELIVERY_SYSTEM = DVBS2
FREQUENCY = 12660197
POLARIZATION = VERTICAL
SYMBOL_RATE = 19194200
INNER_FEC = 1/2
MODULATION = QPSK
```

Get the actual numbers from whoever runs the uplink. Frequency is in kHz here, symbol rate in sym/s. FEC goes as `1/2`, `2/3`, etc - not `FEC_1_2`, that syntax breaks the parser.

## 7. Tune and check lock

```bash
sudo dvbv5-zap -a 0 -c ~/intersputnik.conf -r INTERSPUTNIK -v -l UNIVERSAL
```

`-l UNIVERSAL` is the LNB type - fine for a standard universal LNB.

Good sign:

```
Lock   (0x1f) <signal level> dBm  C/N <value> dB
```

`Lock (0x1f)` means all five lock bits are set (carrier, signal, FEC, sync, demod) - the tuner is locked onto the transponder.

If the frontend is busy:

```bash
sudo fuser -v /dev/dvb/adapter0/frontend0
sudo kill -9 <PID>
```

## 8. Watch it in VLC

```bash
vlc dvb-s://frequency=12660197:srate=19194200:voltage=13:fec=12:modulation=QPSK:delivery_system=DVBS2
```

Same numbers as the config file, just renamed for the URL:
- `frequency` - kHz (not Hz - VLC's own unit conversion here is misleading, don't trust the "too low" warning)
- `srate` - sym/s
- `voltage` - 13 for vertical, 18 for horizontal
- `fec` - `1/2` written as `12`
- `modulation` / `delivery_system` - as-is

## Notes

- No CAM in the CI slot = no decryption. An encrypted channel just won't open, that's expected and not a driver problem.
- If the transponder is carrying something other than a normal video/audio stream (files, IP-over-DVB, whatever), VLC and `dvbv5-zap` will lock the signal fine but you won't get a picture out of it - that needs a custom decoder matching whatever encapsulation the sender used. Ask the provider what that is before assuming the receive chain is broken.
