# ymodem - Receive Files over the Console

## Description

The `ymodem` command receives files over the console it runs on, using the
YMODEM batch protocol. Each file arrives with its name and size, and a
name may contain directories. It is meant for putting files on a device
over its UART: views, configurations, modules, anything that belongs on
the SD card rather than in the firmware.

## Usage

```
ymodem recv [-d DIR]
```

## Options

- `-d DIR` - The directory the files go to (default: the current one).
  Directories in the sent names are created under it as needed.

## Behavior

- The terminal is in raw mode (no echo, no line editing) while the
  transfer runs, and is restored afterwards.
- Until the sender starts, `C` is sent every second, so a sender started
  after the command (`sb` from a terminal program) sees one.
- Blocks of 128 and 1024 bytes with CRC-16 are both accepted. A block
  repeated (an ACK lost) is acknowledged again and written once. A broken
  one is NAKed once the line has been quiet for 100 ms (the rest of it
  read), so the sender's next packet is read from its start: an EOT or CAN
  is only taken there, never a byte of data. 16 errors in a row cancel the
  transfer.
- A sender gone without cancelling leaves the receiver waiting: Ctrl-X
  twice stops it.
- Names with `..` are refused, so nothing is written outside `DIR`.
- At the end it prints how many files and bytes were received.

## Senders

- dmod-boot: `cmake --build . --target upload-files` (its
  `scripts/ymodem_send.py`) types the command and sends the files.
- lrzsz: `sb file...` (or `sb -k` for 1 KB blocks), started in the
  terminal program after `ymodem recv` (e.g. minicom: Ctrl-A S, ymodem).

The UART's receive buffer (dmuart's `rx_ring_size`) holds what comes
while the command is not scheduled: a block is sent at once, so with a
small one (256 bytes is 2.8 ms at 921600 baud) blocks are lost and sent
again - slower, still right. 1 KB blocks want a few KB.

## Exit Codes

- `0` - Every file received
- `-ECANCELED` - The sender cancelled
- `-EIO` - Too many errors, or a file could not be written
- `-EACCES` - A file could not be created
