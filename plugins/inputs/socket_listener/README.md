# Socket Listener Input Plugin

This service plugin listens for messages on sockets (TCP, UDP, Unix or Unixgram)
and parses the packets received in one of the supported
[data formats][data_formats].

⭐ Telegraf v1.3.0
🏷️ network
💻 all

[data_formats]: /docs/DATA_FORMATS_INPUT.md

## Service Input <!-- @/docs/includes/service_input.md -->

This plugin is a service input. Normal plugins gather metrics determined by the
interval setting. Service plugins start a service to listen and wait for
metrics or events to occur. Service plugins have two key differences from
normal plugins:

1. The global or plugin specific `interval` setting may not apply
2. The CLI options of `--test`, `--test-wait`, and `--once` may not produce
   output for this plugin

## Global configuration options <!-- @/docs/includes/plugin_config.md -->

In addition to the plugin-specific configuration settings, plugins support
additional global and plugin configuration settings. These settings are used to
modify metrics, tags, and field or create aliases and configure ordering, etc.
See the [CONFIGURATION.md][CONFIGURATION.md] for more details.

[CONFIGURATION.md]: ../../../docs/CONFIGURATION.md#plugins

## Configuration

```toml @sample.conf
# Generic socket listener capable of handling multiple socket types.
[[inputs.socket_listener]]
  ## URL to listen on
  # service_address = "tcp://:8094"
  # service_address = "tcp://127.0.0.1:http"
  # service_address = "tcp4://:8094"
  # service_address = "tcp6://:8094"
  # service_address = "tcp6://[2001:db8::1]:8094"
  # service_address = "udp://:8094"
  # service_address = "udp4://:8094"
  # service_address = "udp6://:8094"
  # service_address = "unix:///tmp/telegraf.sock"
  # service_address = "unixgram:///tmp/telegraf.sock"
  # service_address = "vsock://cid:port"

  ## Permission for unix sockets (only available on unix sockets)
  ## This setting may not be respected by some platforms. To safely restrict
  ## permissions it is recommended to place the socket into a previously
  ## created directory with the desired permissions.
  ##   ex: socket_mode = "777"
  # socket_mode = ""

  ## Maximum number of concurrent connections (only available on stream sockets like TCP)
  ## Zero means unlimited.
  # max_connections = 0

  ## Read timeout (only available on stream sockets like TCP)
  ## Zero means unlimited.
  # read_timeout = "0s"

  ## Optional TLS configuration (only available on stream sockets like TCP)
  # tls_cert = "/etc/telegraf/cert.pem"
  # tls_key  = "/etc/telegraf/key.pem"
  ## Enables client authentication if set.
  # tls_allowed_cacerts = ["/etc/telegraf/clientca.pem"]

  ## Maximum socket buffer size (in bytes when no unit specified)
  ## For stream sockets, once the buffer fills up, the sender will start
  ## backing up. For datagram sockets, once the buffer fills up, metrics will
  ## start dropping. Defaults to the OS default.
  # read_buffer_size = "64KiB"

  ## Period between keep alive probes (only applies to TCP sockets)
  ## Zero disables keep alive probes. Defaults to the OS configuration.
  # keep_alive_period = "5m"

  ## Content encoding for message payloads
  ## Can be set to "gzip" for compressed payloads or "identity" for no encoding.
  # content_encoding = "identity"

  ## Maximum size of decoded packet (in bytes when no unit specified)
  # max_decompression_size = "500MB"

  ## Message splitting strategy and corresponding settings for stream sockets
  ## (tcp, tcp4, tcp6, unix or unixpacket). The setting is ignored for packet
  ## listeners such as udp.
  ## Available strategies are:
  ##   newline         -- split at newlines (default)
  ##   null            -- split at null bytes
  ##   delimiter       -- split at delimiter byte-sequence in hex-format
  ##                      given in `splitting_delimiter`
  ##   fixed length    -- split after number of bytes given in `splitting_length`
  ##   variable length -- split depending on length information received in the
  ##                      data. The length field information is specified in
  ##                      `splitting_length_field`.
  # splitting_strategy = "newline"

  ## Delimiter used to split received data to messages consumed by the parser.
  ## The delimiter is a hex byte-sequence marking the end of a message
  ## e.g. "0x0D0A", "x0d0a" or "0d0a" marks a Windows line-break (CR LF).
  ## The value is case-insensitive and can be specified with "0x" or "x" prefix
  ## or without.
  ## Note: This setting is only used for splitting_strategy = "delimiter".
  # splitting_delimiter = ""

  ## Fixed length of a message in bytes.
  ## Note: This setting is only used for splitting_strategy = "fixed length".
  # splitting_length = 0

  ## Specification of the length field contained in the data to split messages
  ## with variable length. The specification contains the following fields:
  ##  offset        -- start of length field in bytes from begin of data
  ##  bytes         -- length of length field in bytes
  ##  endianness    -- endianness of the value, either "be" for big endian or
  ##                   "le" for little endian
  ##  header_length -- total length of header to be skipped when passing
  ##                   data on to the parser. If zero (default), the header
  ##                   is passed on to the parser together with the message.
  ## Note: This setting is only used for splitting_strategy = "variable length".
  # splitting_length_field = {offset = 0, bytes = 0, endianness = "be", header_length = 0}

  ## Data format to consume.
  ## Each data format has its own unique set of configuration options, read
  ## more about them here:
  ## https://github.com/influxdata/telegraf/blob/master/docs/DATA_FORMATS_INPUT.md
  # data_format = "influx"
```

### Operating System UDP Buffer Sizes

The `read_buffer_size` config option can be used to adjust the size of the
socket buffer, but this number is limited by OS settings. On Linux,
`read_buffer_size` will default to `rmem_default` and will be capped by
`rmem_max`. On BSD systems, `read_buffer_size` is capped by `maxsockbuf`, and
there is no OS default setting.

Instructions on how to adjust these OS settings are available below.

Some OSes (most notably, Linux) place very restrictive limits on the performance
of UDP protocols. It is _highly_ recommended that you increase these OS limits
to at least 8MB before trying to run large amounts of UDP traffic to your
instance.  8MB is just a recommendation, and can be adjusted higher.

#### Linux

Check the current UDP/IP receive buffer limit and default by using the following
commands:

```sh
sysctl net.core.rmem_max
sysctl net.core.rmem_default
```

If the values are less than 8388608 bytes you should add the following lines to
the /etc/sysctl.conf file:

```text
net.core.rmem_max=8388608
net.core.rmem_default=8388608
```

Changes to /etc/sysctl.conf do not take effect until reboot.
To update the values immediately, type the following commands as root:

```sh
sysctl -w net.core.rmem_max=8388608
sysctl -w net.core.rmem_default=8388608
```

#### BSD/Darwin

On BSD/Darwin systems you need to add about a 15% padding to the kernel limit
socket buffer. Meaning if you want an 8MB buffer (8388608 bytes) you need to set
the kernel limit to `8388608*1.15 = 9646900`. This is not documented anywhere
but can be seen [in the kernel source code][kernel_source].

Check the current UDP/IP buffer limit by typing the following command:

```sh
sysctl kern.ipc.maxsockbuf
```

If the value is less than 9646900 bytes you should add the following lines
to the /etc/sysctl.conf file (create it if necessary):

```text
kern.ipc.maxsockbuf=9646900
```

Changes to /etc/sysctl.conf do not take effect until reboot.
To update the values immediately, type the following command as root:

```sh
sysctl -w kern.ipc.maxsockbuf=9646900
```

[kernel_source]: https://github.com/freebsd/freebsd/blob/master/sys/kern/uipc_sockbuf.c#L63-L64

## Metrics

The plugin accepts arbitrary input and parses it according to the `data_format`
setting. There is no predefined metric format.

## Example Output

There is no predefined metric format, so output depends on plugin input.
