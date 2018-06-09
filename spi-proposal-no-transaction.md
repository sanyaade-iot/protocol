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

A `SPI_BEGIN` command is used to initialize the SPI bus. Up to 8 SPI ports
are supported using the `channel` parameter.

The `SPI_DEVICE_CONFIG` is used to configure each attached SPI device.

There are 3 ways to send and receive data from the SPI slave device:

1. `SPI_TRANSFER` For each word written a word is read simultaneously.
2. `SPI_WRITE` Only write data (ignore any data returned by the slave device).
3. `SPI_READ` Only read data, writing `0` for each word to be read.

A `SPI_REPLY` is used to send requested data back to the client application when
either a `SPI_TRANSFER` mode or `SPI_READ` command is sent.

A `SPI_END` command disables the SPI bus for the specified channel.

### SPI_BEGIN

Required for platforms that require SPI bus initialization, such as Arduino.
Optional if initialization is automatic (some Linux-based platforms for
example).

Use `SPI_BEGIN` to initialize the SPI bus. Up to 8 SPI ports are supported,
where each port is identified by a `channel` number (0-7).

`SPI_BEGIN` must be called at least once before sending any of the other
commands.

`channel` is used to identify which SPI bus is used in the case that a board
has multiple ports (SPI, SPI1, SPI2, etc). Many boards only have one port so the
`channel` value will most often be `0`.

```
0:  START_SYSEX
1:  SPI_DATA              (0x68)
2:  SPI_BEGIN             (0x00)
3:  channel               (HW supports multiple SPI ports. range = 0-7, default = 0)
4:  END_SYSEX
```

### SPI_DEVICE_CONFIG

Send this command once for each attached SPI device to initialize it before use.

The `deviceId` may either be used as a specific identifier (Linux) or as an
arbitrary identifier (Arduino). The `deviceId` value range is from 0 to 15 and
can be specified separately for each SPI channel. The `deviceId` will also be
returned with the channel for each `SPI_REPLY` command so it is clear which
device the data corresponds to.

A chip select pin (`csPin`) can optionally be specified. This pin will be
controlled per the rules specified in `csPinOptions`. For uncommon use cases
of CS or other required HW pins per certain SPI devices, it is better to control
them separately by setting `CS_PIN_CONTROL` to 0 (disable) and using Firmata
`DIGITAL_MESSAGE` commands to control the CS pin state. If `CS_PIN_CONTROL` is
enabled, the `csPin` number must also be specified.

```
0:  START_SYSEX
1:  SPI_DATA              (0x68)
2:  SPI_DEVICE_CONFIG     (0x01)
3:  deviceId | channel    (bits 3-6: deviceId, bits 0-2: channel)
4:  dataMode | bitOrder   (bits 1-2: dataMode (0-3), bit 0: bitOrder)
5:  maxSpeed              (bits 0 - 6)
6:  maxSpeed              (bits 7 - 14)
7:  maxSpeed              (bits 15 - 21)
8:  maxSpeed              (bits 22 - 28)
9:  maxSpeed              (bits 29 - 32)
10: wordSize              (0 = DEFAULT = 8-bits, 1 = 1-bit, 2 = 2 bits, etc)
11: csPinOptions          bit 0: CS_PIN_CONTROL (0 = disable
                                                 1 = enable (default))
                          bit 1: CS_ACTIVE_STATE (0 = Active LOW (default)
                                                  1 = Active HIGH)
                          bits 2-6: reserved for future options
12: csPin                 (0-127) The chip select pin number
13: END_SYSEX
```

#### bitOrder
```
LSBFIRST = 0
MSBFIRST = 1 (default)
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

#### csPinOptions / csPin

To enable automatic control of the csPin, set bit 0 (`CS_PIN_CONTROL`) to 1
(enable). To manually control the csPin state with `DIGITAL_MESSAGE` commands,
set `CS_PIN_CONTROL` to 0 (disable).

The default behavior is to pull the CS pin LOW at the beginning of a transaction
and toggle it at the end of the transaction. Setting bit 1 (`CS_ACTIVE_STATE`)
will pull the CS pin HIGH at the start of the transaction.

If `CS_PIN_CONTROL` is set to 0 (disable), the csPin parameter is ignored. Set
the value to zero. The user will then need to manually control the csPin state
using `DIGITAL_MESSAGE` commands.

### SPI_TRANSFER

Full-duplex write/read transfer. This is the normal SPI transfer mode, a word
must be written for every word read. Reply is sent via `SPI_REPLY`.

The user can continue sending as many transfer command as necessary, even
alternating between `SPI_TRANSFER`, `SPI_WRITE` and `SPI_READ` within a single
SPI transaction.

`requestId` is used in the request messages `SPI_TRANSFER`, `SPI_WRITE` and
`SPI_READ` and in the reply message `SPI_REPLY`. Its purpose is to ensure that
the SPI_REPLY message matches the request. For each request message, increment
a single 7-bit requestId value, rolling it over to 0 when > 127.

`csPinChange` is used to control the csPin at the end of the transfer. By
default the csPin will be deselected (the csPin will be toggled). However, to
prevent deselection to enable back-to-back transfers for example, set
`csPinChange` to 0 and the pin state won't be affected.

If `CS_PIN_CONTROL` is enabled, then the csPin active state will be set when
the `SPI_TRANSFER` command is received. It will only be deselected (toggled) at
the end of the transfer if `csPinChange` is set to 1.

```
0:  START_SYSEX
1:  SPI_DATA              (0x68)
2:  SPI_TRANSFER          (0x02)
3:  deviceId | channel    (bits 3-6: deviceId, bits 0-2: channel)
4:  requestId             (0-127) // increment for each call
5:  csPinChange           (0 = don't deselect csPin
                           1 = deselect csPin (default))
6.  numWords              (0-127: number of words to transfer)
7:  data 0                (bits 0-6)
8:  data 0                (bits 7-14 if word size if word size > 7 && < 15)
9:  data 0                (if word size > 14)
...                       up to numWords * (wordSize / 7)
N:  END_SYSEX
```

### SPI_WRITE

Only write data, ignoring any data returned by the slave device. No `SPI_REPLY`
command is sent.

Provided as a convenience. The same can be accomplished using `SPI_TRANSFRER`
and ignoring the `SPI_REPLY` command.

If `CS_PIN_CONTROL` is enabled, then the csPin active state will be set when
the `SPI_WRITE` command is received. It will only be deselected (toggled) at the
end of the write if `csPinChange` is set to 1.

```
0:  START_SYSEX
1:  SPI_DATA              (0x68)
2:  SPI_WRITE             (0x03)
3:  deviceId | channel    (bits 3-6: deviceId, bits 0-2: channel)
4:  requestId             (0-127) // increment for each call
5:  csPinChange           (0 = no change, 1 = toggle)
6.  numWords              (0-127: number of words to write)
7:  data 0                (bits 0-6)
8:  data 0                (bits 7-14 if word size if word size > 7 && < 15)
9:  data 0                (if word size > 14)
...                       up to numWords * (wordSize / 7)
N:  END_SYSEX
```

### SPI_READ

Only read data, writing `0` for each word to be read. Reply is sent via
`SPI_REPLY`.

Provided as a convenience. The same can be accomplished using `SPI_TRANSFRER`
and sending a `0` for each byte to be read.

If `CS_PIN_CONTROL` is enabled, then the csPin active state will be set when
the `SPI_READ` command is received. It will only be deselected (toggled) at the
end of the read if `csPinChange` is set to 1.

```
0:  START_SYSEX
1:  SPI_DATA              (0x68)
2:  SPI_WRITE             (0x04)
3:  deviceId | channel    (bits 3-6: deviceId, bits 0-2: channel)
4:  requestId             (0-127)  // increment for each call
5:  csPinChange           (0 = no change, 1 = toggle)
6.  numWords              (0-127: number of words to read)
7:  END_SYSEX
```

### SPI_REPLY

An array of data received from the SPI slave device in response to a
`SPI_TRANSFER` or `SPI_READ` command.

```
0:  START_SYSEX
1:  SPI_DATA              (0x68)
2:  SPI_REPLY             (0x05)
3:  deviceId | channel    (bits 3-6: deviceId, bits 0-2: channel)
4:  requestId             (0-127) // must match the ID from the request
5:  numWords              (0-127: number of words in the reply)
6:  data 0                (bits 0-6)
7:  data 0                (bits 7-14 if word size if word size > 7 && < 15)
8:  data 0                (if word size > 14)
...                       up to numWords * (wordSize / 7)
N:  END_SYSEX
```

### SPI_END

Call to release SPI hardware send before quitting a Firmata client application.

```
0:  START_SYSEX
1:  SPI_DATA              (0x68)
2:  SPI_END               (0x06)
3:  channel               (HW supports multiple SPI ports. range = 0-7, default = 0)
4:  END_SYSEX
```