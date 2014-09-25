Floppy Controller Driver
========================


Overview
---------
The floppy disk controller (`FDC`) is a device that controls access to floppy
medium. There are many different chips available, but the most common is the
`82077AA` which has been produced since 1991.

Like many x86 devices, the floppy subsystem is ugly. Status bits are often
inverted (such as a bit indicating if `DMA` transfer is *NOT* enabled) and there
is often a few different ways to approach the same task.

The `FDC` supports up to two cables with two drives on each for a total of 4
drives, but this is not always the case.

Floppy medium is a magnetic disk that is controlled my mechanical motors and
a head that is used to seek, read, and write and as a result this medium is
unreliable. To make matters worse, the `FDC` has unreliable communication, often
requiring up to 3 suggested read or write commands to ensure data has been
transfered correctly.

Like most hardware, specifically mechanical implementation, there are timing
issues. In this case it is particularly bad, the `FDC` does not inform the
kernel when a motor is up to speed, when the controller can accept commands,
or when the controller has data to read/write (depending on I/O settings). The
aforementioned timing issues are only exacerbated on modern hardware that
performs orders of magnitude faster than most controller's original design.

Overall is not recommended to pursue a `FDC` driver when there are plenty of
sane alternatives such as Parallel ATA (`PATA`).


Registers
---------
The `FDC` is programmed through 9 registers accessed through offsets from the
base I/O port at `0x03F0`:

	STATUS_REG_A            0x0000
	STATUS_REG_B            0x0001
	DIGITAL_OUTPUT_REG      0x0002
	TAPE_DRIVE_REGISTER     0x0003
	MAIN_STATUS_REG         0x0004
	DATA_RATE_SELECT_REG    0x0004
	DATA_REGISTER           0x0005
	DIGITAL_INPUT_REG       0x0007
	CONFIG_CONTROL_REG      0x0007

All commands, parameters, result codes, and data transfers go through the FIFO
port (`DATA_REGISTER`). The `MAIN_STATUS_REG` contains status bitflags, that
must be checked before reading/writing each byte. `DIGITAL_OUTPUT_REG` controls
the motors, selection process, and reset process. Other registers contain often
unnecessary information.


### Digital Output Register
The digital output register (`DOR`) is a byte who's bits controls controller
functionality:

	MOTD  (bit 7) - Drive 3 motor ON
	MOTC  (bit 6) - Drive 2 motor ON
	MOTB  (bit 5) - Drive 1 motor ON
	MOTA  (bit 4) - Drive 0 motor ON
	IRQ   (bit 3) - IRQ/DMA mode ON
	RESET (bit 2) - NOT enter reset mode
	DSEL1 (bit 1) - Select drive 1
	DSEL0 (bit 0) - Select drive 0

To set the `DOR` to turn drive 0's motor on, select drive 0, and enable normal
operation without `DMA`: 00010101 (0x15)


### Main Status Register
The main status register (`MSR`) is a byte who's bits indicates controller
status:

	RQM  (bit 7) - Set if I/O exchange is possible or needed
	DIO  (bit 6) - Set if `DATA_REGISTER` expects IN opcode
	NDMA (bit 5) - Set in execution phase of PIO read/write commands
	CB   (bit 4) - Command busy, set when a command is executing
	ACTD (bit 3) - Drive 3 is seeking
	ACTC (bit 2) - Drive 2 is seeking
	ACTB (bit 1) - Drive 1 is seeking
	ACTA (bit 0) - Drive 0 is seeking


### Digital Input Register
Contains at bit 7 a value that indicates if the floppy door was open or closed.


Initialization
--------------
The initial state of the `FDC` is unknown, so initialization must be done in
order to use it:

	Send a `VERSION` command to the controller, verify result byte is 0x90
	Do a configure procedure
	Do a reset procedure
	Do a drive selection procedure
	Do another configure procedure
	Send a `RECALIBRATE` command to each drive


### Configure Procedure
	Send `CONFIGURE` command
	Send 0x00
	Send parameter byte
	Send 0x00

parameter byte = (`seek_enabled` << 6) | (`fifo_disabled` << 5) |
(`polling_disabled` << 4) | (`threshold` - 1)

A good `threshold` value is 15 and is the amount of bytes to wait between
interrupts


### Reset Procedure
	Disable controller by writing 0x00 to `DOR`
	Write desired `DOR` configuration to `DOR`


### Drive Selection
	Send 0x00 (1.44MB floppy data rate) to configuration control register
	Ensure the `DOR` is set correctly (which it should be)


Command Subsystem
-----------------
Communication is done through the command subsystem, which is essentially
checking if the subsystem is ready, writing to the `DATA_REGISTER` port, and
checking results.

Here is a list of commands that are useful for basic functionality:

	FIX_DRIVE_DATA          0x03
	CHECK_DRIVE_STATUS      0x04
	CALIBRATE_DRIVE         0x07
	CHECK_INTERRUPT_STATUS  0x08
	FORMAT_TRACK            0x4D
	READ_SECTOR             0x66
	READ_DELETE_SECTOR      0xCC
	READ_SECTOR_ID          0x4A
	READ_TRACK              0x42
	SEEK_TRACK              0x0F
	WRITE_SECTOR            0xC5
	WRITE_DELETE_SECTOR     0xC9
	VERSION                 0x10
	CONFIGURE               0x13

It is also noted that a few option bits may be required when issuing read,
write, format and verify commands:

	MULTI_TRACK              0x80
	MAGNETIC_ENCODING        0x40
	SKIP_MODE                0x20

These masks are simply ORed with aforementioned commands. `MAGNETIC_ENCODING`
should always be set for all read, write, format and verify commands.

### Status Registers
There 3 registers that are not available by reading an offset from the
controller's base address. These are registers `st0`, `st1`, and `st2`.

The top 2 bits (7, 6) of `st0` are set after a reset with polling on. If either
bit is set at any other time then an error has occurred. Bit 5 is set after
a calibrate or seek.

Bit 7 of `st1` is set if the floppy ran out of sectors to complete a read or
write. Bit 4 is set if the driver is operating too slow to transfer data. Bit 1
is set if the media is write protected. The rest of the bits indicate various
errors.

All bits of `st2` indicate various errors that are hardware specific.

### Sending Commands

	Read `MSR`
	Verify `RQM` and `DIO` are set correctly
	Send command to `DATA_REGISTER`
	For each parameter:
		Read `MSR` until `RQM` is set, then send parameter
		Read `MSR`, if `NDMA` is set skip to result phase
	Result phase: read expected results bytes (if any) and check for errors


### Reading and Writing
	Turn motor on
	Send read or write command with correct bitflags
	Send (`head_number` << 2) | `drive_number`
	Send `track_number`
	Send `head_number`
	Send `sector_number`
	Send 0x02
	Send sector count to transfer
	Send 0x1B
	Send 0xFF
	Read `st0`
	Read `st1`
	Read `st2`
	Read `tracker_number`
	Read `ending_head_number`
	Read `ending_sector_number`
	Read 0x02


Function Declarations
---------------------

	/*
	  Initializes floppy controller for drive 0
	  Params: ctrlr  - floppy controller base address
	          floppy - floppy structure
	*/
	void _floppy_init(floppy_controller_t ctrlr, floppy_parameters_t *floppy);


	/*
	  Seeks to a given track on an initialized floppy controller, drive 0
	  Params: ctrlr  - floppy controller base address
	          floppy - floppy structure
	          track  - track number 
	*/
	void _floppy_seek(floppy_controller_t ctrlr, floppy_parameters_t *floppy,
	                  floppy_track_t track);


	/*
	  Reads a given track on an initialized floppy controller, drive 0
	  Params: ctrlr  - floppy controller base address
	          floppy - floppy structure
	          data   - buffer to read track into
	          track  - track number 
	*/
	void _floppy_read(floppy_controller_t ctrlr, floppy_parameters_t *floppy,
	                  unsigned char *data, floppy_track_t track);


	/*
	  Writes a given track on an initialized floppy controller, drive 0
	  Params: ctrlr  - floppy controller base address
	          floppy - floppy structure
	          data   - buffer to write to track
	          track  - track number 
	*/
	void _floppy_write(floppy_controller_t ctrlr, floppy_parameters_t *floppy,
	                   unsigned char *data, floppy_track_t track);


	/*
	  Formats a given track on an initialized floppy controller, drive 0
	  Params: ctrlr  - floppy controller base address
	          floppy - floppy structure
	          track  - track number 
	*/
	void _floppy_format(floppy_controller_t ctrlr, floppy_parameters_t *floppy,
	                    floppy_track_t track);