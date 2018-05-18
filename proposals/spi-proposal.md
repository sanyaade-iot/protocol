SPI (Proposal)
===

A proposal for a SPI protocol for Firmata.

SPI is tricky to add to Firmata in a generic way since it is a fairly loose
standard. There are variations in number of bits written and read, how the CS
pin is used (if at all), use of other pins, etc. This proposal attempts to
accommodate uses cases beyond the common sequence of:

1. set cs pin LOW
2. write/read 1 or more words
3. set cs Pin HIGH
4. return data read

### Overview

A `SPI_BEGIN` command is used to initialize the SPI bus. Up to 4 SPI ports
are supported using the `channel` parameter.

The `SPI_DEVICE_CONFIG` is used to configure each attached SPI device.

In the same spirit as the Arduino SPI library, a `SPI_BEGIN_TRANSACTION` command
and a `SPI_END_TRANSACTON` command are available to frame data transfers related
to a specific SPI device. This is useful in Arduino applications by reinforcing
that arbitrary reads and writes can't occur between 2 or more SPI devices
without being part of a transaction (since each device may have a different
clock speed, data mode, bit order, csPin, etc).

There are 3 ways to send and receive data from the SPI slave device:

1. `SPI_TRANSFER` For each word written a word is read simultaneously.
2. `SPI_WRITE` Only write data (ignore any data returned by the slave device).
3. `SPI_READ` Only read data, writing `0` for each word to be read.

A `SPI_REPLY` is used to send requested data back to the client application when
either a `SPI_TRANSFER` mode or `SPI_READ` command is sent.

A `SPI_END` command disables the SPI bus for the specified channel.

### SPI_BEGIN

Use `SPI_BEGIN` to initialize the SPI bus. Up to 4 SPI ports are supported,
where each port is identified by a `channel` number (0-3).

`SPI_BEGIN` must be called at least once before sending any of the other
commands.

`channel` is used to identify which SPI bus is used in the case that a board
has multiple ports (SPI, SPI1, SPI2, etc). Many boards only have one port so the
`channel` value will most often be `0`.

```
0:  START_SYSEX
1:  SPI_DATA              (0x68)
2:  SPI_BEGIN             (0x00)
3:  channel               (HW supports multiple SPI ports. range = 0-3, default = 0)
4:  END_SYSEX
```

### SPI_DEVICE_CONFIG

Send this command once for each attached SPI device to initialize it before use.

The `deviceId` may either be used as a specific identifier (Linux) or as an
arbitrary identifier (Arduino). The `deviceId` value range is from 0 to 31 and
can be specified separately for each SPI channel. The `deviceId` will also be
returned with the channel for each `SPI_REPLY` command so it is clear which
device the data corresponds to.

A chip select pin (`csPin`) can optionally be specified. This pin will be
controlled per the rules specified in `csPinOptions`. For uncommon use cases
of CS or other required HW pins per certain SPI devices, it is better to control
them separately by not specifying a CS pin and instead using Firmata
`DIGITAL_MESSAGE` to control the CS pin state. If a CS pin is specified, the
`csPinOptions` for that pin must also be specified.

```
0:  START_SYSEX
1:  SPI_DATA              (0x68)
2:  SPI_DEVICE_CONFIG     (0x01)
3:  deviceId | channel    (bits 2-6: deviceId, bits 0-1: channel)
4:  dataMode | bitOrder   (bits 1-2: dataMode (0-3), bit 0: bitOrder)
5:  maxSpeed              (bits 0 - 6)
6:  maxSpeed              (bits 7 - 14)
7:  maxSpeed              (bits 15 - 21)
8:  maxSpeed              (bits 22 - 28)
9:  maxSpeed              (bits 29 - 32)
10: wordSize              (0 = DEFAULT = 8-bits, 1 = 1-bit, 2 = 2 bits, etc)
11: csPin                 [optional] (0-127) The chip select pin number
12: csPinOptions          bit 0: CS_ACTIVE_STATE (0 = Active LOW (default)
                                                  1 = Active HIGH)
                          bits 1-6: reserved for future options
11|13: END_SYSEX
```

#### bitOrder
```
LSBFIRST = 0 (default)
MSBFIRST = 1
```

#### dataMode

| mode    | clock polarity (CPOL) | clock phase (CPHA) |
| --------|-----------------------|--------------------|
| 0       | 0                     | 0                  |
| 1       | 0                     | 1                  |
| 2       | 1                     | 0                  |
| 3       | 1                     | 1                  |

#### maxSpeed

The maximum speed of communication with the SPI device. For a SPI device rated
up to 5 MHz, use 5000000.

*For platforms that use a clock divider instead of a speed, pass the clock
divider value instead.* 

#### wordSize

The size of a `word` in bits. Typically 8-bits (default). 0 = DEFAULT = 8-bits,
1 = 1 bit, 2 = 2 bits, etc (limit is TBD).

#### csPin / csPinOptions

Specify a CS (chip select) pin to automatically toggle the pin rather than
having to send a separate `DIGITAL_MESSAGE` command at the beginning and end of
every transaction.

The default behavior is to pull the CS pin LOW at the beginning of a transaction
and toggle it at the end of the transaction. Setting bit 0 (`CS_ACTIVE_STATE`)
will pull the CS pin HIGH at the start of the transaction.

### SPI_BEGIN_TRANSACTION

Required for platforms that support the concept of SPI transactions, such as
Arduino. Optional if transactions are not supported (some Linux-based
platforms for example).

The `deviceId` parameter is used to identify which device on the specified
channel the transaction belongs two. 

If a CS pin is specified for the device in the `SPI_DEVICE_CONFIG` command, the
pin will automatically be set to the `CS_ACTIVE_STATE` at the beginning of the
transaction.

Send the `SPI_END_TRANSACTION` command at the end of the transaction.

```
0:  START_SYSEX
1:  SPI_DATA              (0x68)
2:  SPI_BEGIN_TRANSACTION (0x02)
3:  deviceId | channel    (bits 2-6: deviceId, bits 0-1: channel)
4:  END_SYSEX
```

### SPI_END_TRANSACTION

Required for platforms that support the concept of SPI transactions, such as
Arduino. Optional if transactions are not supported (some Linux-based
platforms for example).

The `deviceId` parameter is used to identify which device on the specified
channel the transaction belongs two. 

If a CS pin is specified for the device in the `SPI_DEVICE_CONFIG` command, the
pin will automatically be toggled at the end of the transaction.

```
0:  START_SYSEX
1:  SPI_DATA              (0x68)
2:  SPI_END_TRANSACTION   (0x03)
3:  deviceId | channel    (bits 2-6: deviceId, bits 0-1: channel)
4:  END_SYSEX
```

### SPI_TRANSFER

Full-duplex write/read transfer. This is the normal SPI transfer mode, a word
must be written for every word read. Reply is sent via `SPI_REPLY`.

The user can continue sending as many transfer command as necessary, even
alternating between `SPI_TRANSFER`, `SPI_WRITE` and `SPI_READ` within a single
SPI transaction.

```
0:  START_SYSEX
1:  SPI_DATA              (0x68)
2:  SPI_TRANSFER          (0x04)
3:  deviceId | channel    (bits 2-6: deviceId, bits 0-1: channel)
4.  numWords              (number of words to transfer)
5:  data 0                (bits 0-6)
6:  data 0                (bits 7-14 if word size if word size > 7 && < 15)
7:  data 0                (if word size > 14)
...                       up to numWords * (wordSize / 7)
N:  END_SYSEX
```

### SPI_WRITE

Only write data, ignoring any data returned by the slave device. No `SPI_REPLY`
command is sent.

Provided as a convenience. The same can be accomplished using `SPI_TRANSFRER`
and ignoring the `SPI_REPLY` command.

```
0:  START_SYSEX
1:  SPI_DATA              (0x68)
2:  SPI_WRITE             (0x05)
3:  deviceId | channel    (bits 2-6: deviceId, bits 0-1: channel)
4.  numWords              (number of words to write)
5:  data 0                (bits 0-6)
6:  data 0                (bits 7-14 if word size if word size > 7 && < 15)
7:  data 0                (if word size > 14)
...                       up to numWords * (wordSize / 7)
N:  END_SYSEX
```

### SPI_READ

Only read data, writing `0` for each word to be read. Reply is sent via
`SPI_REPLY`.

Provided as a convenience. The same can be accomplished using `SPI_TRANSFRER`
and sending a `0` for each byte to be read.

```
0:  START_SYSEX
1:  SPI_DATA              (0x68)
2:  SPI_WRITE             (0x06)
3:  deviceId | channel    (bits 2-6: deviceId, bits 0-1: channel)
4.  numWords              (number of words to read)
5:  END_SYSEX
```

### SPI_REPLY

An array of data received from the SPI slave device in response to a
`SPI_TRANSFER` or `SPI_READ` command.

```
0:  START_SYSEX
1:  SPI_DATA              (0x68)
2:  SPI_REPLY             (0x07)
3:  deviceId | channel    (bits 2-6: deviceId, bits 0-1: channel)
4:  numWords
5:  data 0                (bits 0-6)
6:  data 0                (bits 7-14 if word size if word size > 7 && < 15)
7:  data 0                (if word size > 14)
...                       up to numWords * (wordSize / 7)
N:  END_SYSEX
```

### SPI_END

Call to release SPI hardware send before quitting a Firmata client
application.

```
0:  START_SYSEX
1:  SPI_DATA              (0x68)
2:  SPI_END               (0x08)
3:  channel               (HW supports multiple SPI ports. range = 0-3, default = 0)
4:  END_SYSEX
```