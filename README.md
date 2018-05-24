# EasyNIC: an easy-to-use host interface for network cards

EasyNIC is the specification of a hacker-friendly interface between a
computer and a network interface card (NIC).

Design goals:
- Make device driver development easy.
- Provide a minimal "high-speed serial port" operating mode.
- Support 100G and beyond (use PCIe bandwidth efficiently.)
- Support optional extensions (in later versions.)

EasyNIC is inspired by the success of RISC-V.

# Transmit

The host provides packets to the NIC in variable-size *blocks* that
each contain a series of packets. Each packet is represented by a
16-bit `LENGTH` field followed by `PAYLOAD` of the corresponding
size. The end of the block is indicated with a `LENGTH` of zero.

Note: The Ethernet FCS is automatically calculated and appended.

Registers:

    TX_BLOCK_SEND [Write]:

      Write the 64-bit address of a block and enqueue it for
      transmission.

      Multiple writes cause blocks to be transmitted in FIFO order.

      This register must only be written when TX_BLOCK_AVAIL and
      TX_READY read non-zero.

    TX_BLOCK_AVAIL [Read]

      Read how many more blocks can be enqueued at the current time.

      The value is determined strictly as follows:

      - Initialized to a device-specific constant (e.g. 1024);
      - Decreases by 1 when a new block is enqueued;
      - Increases by 1 when a block completes.

      The host can poll this value to track which blocks have been
      completed.

      This register must only be read when TX_READY reads non-zero.

    TX_RESET [Write]

      Write the value 1 to initiate an asynchronous reset of the device
      transmit state. This potentially discards the contents of enqueued
      blocks.

      Completion of the reset can be detected via TX_READY. Upon
      completion the value of TX_BLOCK_AVAIL will have been reset to its
      initial value.

    TX_READY [Read]

      Read whether the device is ready to transmit data.

      The value is determined strictly as follows:

      - Initialized to 0 on startup.
      - Transitions to 1 when ready.
      - Transitions back to 0 when a reset is requested.

      The host driver should therefore poll for a non-zero value during
      initialization and after each asynchronous reset.

# Receive
# Diagnostics

