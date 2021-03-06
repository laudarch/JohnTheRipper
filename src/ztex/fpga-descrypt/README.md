## Overview

- Theoretical performance for 4 FPGAs (if its input buffer always has data) on 1 hash at default frequency is 828 Mc/s. With more than 255 hashes/salt there's some decrease in performance. It supports up to 2,047 hashes/salt.
- The design has 24 cores. Each core includes two main units: a pipeline of DES rounds and a comparator. Each of two main units is supplied with its own clock.
- Startup values for clocks are 220,160 MHz. Runtime frequency adjustment is available. If you're running against few hashes/salt then comparators aren't bottlenecks, an increase of frequency on the 2nd clock does not increase performance.
- Device utilization summary: 58% LUTs, 51% BRAMs, 29% FFs, 1% DSPs.


## Communication framework ("Inouttraffic") details

The FPGA application is built on the framework that performs following tasks:
- Communication with USB device controller on a multi-FPGA board;
- Temporary storage of I/O data in input and output buffers;
- Extraction of input application-level data packets from input buffer, checksum verification;
- Creation of output packets from application data, checksum calculation;
- Handling of clocks and reset.

There's an onboard generator. Loaded with configuration and fed with template list, it's able to generate 1 candidate password each cycle.


## Design RTL details

Candidate passwords move from the generator to arbiter unit.
- Each candidate has IDs, that's a requirement for proper accounting; totally 16-bit packet ID, 16-bit word ID and 32-bit ID from the generator result in 64 bits of IDs, that's more than a candidate password itself (56 bit).
- When candidates are sent to cores, IDs are saved into arbiter's memory. In case where a comparator detects equality, IDs are sent for output in CMP_EQUAL packet.
- Host software is expected to reconstruct the candidate password base on IDs, perform hashing and then perform comparison. False positives happen often because FPGA compares only 35 lower bits. There was an idea to simplify host software, send computed result to the host, if that was implemented that would have consumed additional virtual circuitry.
- A total number of computed candidates is also accounted. It sends PROCESSING_DONE packet after it finishes processing of an input packet.


## Design Placement and Routing details

- Substantial attention was paid for optimal placement of individual components. While tools can do that fully automately, it's assumed if developer watches them more then results are better.
- Cores are placed in 4 columns with spacing in between for 2 thin vertical columns (2-3 CLBs wide) for internal data bus. Each one of 24 cores is manually allocated an individual area (typically 30x21 or 30x22 CLBs, containing no less than 4 2K-BRAMs).


## Notes on creation of the bitstream using ISE 14.5

- descrypt_core module is synthesized in separate ISE project. Then the resulting *.ngc file is used in the "main" ISE project. When it sees a declaration of an empty ("blackbox") module then it searches for NGC file in directories specified with -sd option.
- Partitioning is used. In xpartition.pxml file they are listed 24 descrypt_core's in ""/ztex_inouttraffic/pkt_comm/arbiter/wrapper_gen[*].wrapper/core_gen[*].core"" partitions. Please read Hierarchial Design Methodology Guide for details.
- Multi-Pass Place & Route is used. The design allows to declare some cores as "dummy". It was found practical to try 4-6 cores each Map and Place & Route run while the rest of the cores are declared as dummy ones. After such a run, Post Place and Route Static Timing Report is used to find which cores don't violate timing. That cores are marked as "good", declared as "dummy" for the next run, result files containing "good" placement are saved into a separate "export" directory. Typically up to 50% cores in a run are OK, while some cores continue to violate timing after many attempts. In such a case FPGA editor is used to see what's wrong and to make design decisions such as to review area constraints.
- On the final run that includes bitstream generation, all partitions are already built, marked as "import" in xpartition.pxml file.
- If an intermediate check is desired, it's possible to generate a bitstream with a number of "dummy" cores, that will work with reduced performance.


## Further improvements.

It's clear the design doesn't fully use hardware resources (Please see ztex_inouttraffic_summary.html).
There's a bottleneck where it generates and distributes 1 candidate password each cycle. So an addition of more cores doesn't increase performance.

Following possible improvements were identified:
- Deploy two generators, each connected to an arbiter and distribution network, to overcome a limit of 1 candidate/cycle for each IC (total ~880 Mc/s)
- Increase pipeline length from 16 to 80 stages. That increases size of a core and greatly reduces per-core overhead
- More effective comparator design

