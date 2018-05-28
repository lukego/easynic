# EasyNIC: an easy-to-use host interface for network cards

EasyNIC is the specification of a hacker-friendly interface between a
computer and a network interface card (NIC).

Design goals:
- Make device driver development easy.
- Provide a minimal "high-speed serial port" operating mode.
- Support 100G and beyond (use PCIe bandwidth efficiently.)
- Support optional extensions (in later versions.)

EasyNIC is inspired by the success of RISC-V.

# Transmit and Receive

The host provides one buffer for transmit and one for receive. Packets
are prefixed with a 16-bit length header and stored back-to-back. The
buffers are [rings of bytes](https://en.wikipedia.org/wiki/Circular_buffer)
i.e. they automatically wrap around once the last byte has been used. The
buffers are treated as continuous and no space is used for alignment.

The host writes its updated cursor positions to the device via
registers. The device writes its updated cursor positions to the host
using DMA. (The host does not read the device state via registers
because that would require a high-latency blocking read over PCIe.)

Note: The Ethernet FCS is automatically added and removed by the
device. If an incoming frame has an invalid FCS then it is
automatically dropped.

Registers:

    TX_START [write]
    RX_START [write]

      The address of the first byte of the buffer.

    TX_SIZE [write]
    RX_SIZE [write]

      The total size of the buffer in bytes.

    TX_CURSOR [write]
    RX_CURSOR [write]

      The current buffer position from the host perspective i.e. the
      offset from the start of the buffer at which the next packet
      will be written (transmit) or read (receive).

    TXRX_STATUS [write]

      The address where the device will store up-to-date hardware
      transmit and receive cursor positions in host memory.

      The device writes a 16-byte record to this address comprised of
      the 64-bit transmit cursor followed by the 64-bit receive
      cursor. The cursors specify the offsets from the start of the
      buffer where the next packet will be fetched (transmit) and
      stored (receive).

      The device is expected to update this record as rapidly as
      possible without compromising overall throughput.

    TXRX_RESET [write]

      Write the value 1 to initiate an asynchronous reset of the
      device transmit/receive state. This potentially discards the
      contents of enqueued blocks.

      Completion of the reset can be detected via TXRX_READY. Upon
      completion the value of TX_BLOCK_AVAIL will have been reset to
      its initial value.

    TXRX_READY [read]

      Read whether the device is ready to transmit and receive data.

      The value is determined strictly as follows:

      - Initialized to 0 on startup.
      - Transitions to 1 when ready.
      - Transitions back to 0 when a reset is requested.

      The host driver should therefore poll for a non-zero value during
      initialization and after each asynchronous reset.

# Receive
# Diagnostics

