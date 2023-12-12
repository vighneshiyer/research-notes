# Cache Functional Warmup Notes

## Chipyard SoC Cache Configuration

### Dcache

- Configuration
    - 64B block size
    - 16384B total cache size
    - 64 sets = 4 ways
    - 6 bits for offset, 6 bits for set index, 20 bits for tag = 32 bits (we have a 32-bit physically addressable space)
- There is a 'resetting' sequence for the cache where all the tag bits are zeroed out
    - This isn't implemented with a reset net in order to reduce reset fanout and to pack the valid bits inside the SRAM
    - We should attempt to force state during 'resetting' and release right after 'resetting' falls

#### Tag Array

- There are no explicit valid bits
- The tag array contains physical addresses
- Tag array is physically arranged with
    - 4 banks (for 4 ways)
    - Each bank has 64 addresses (6 address bits) for each set
    - Each set holds tags for 4 ways and each tag is 22 bits
- Tag is stored as the raw tag bits + 2 extra bits for coherency
    - 20 bits of raw tag + 2 bits of metadata
    - Upper 2 bits indicate coherency state (see ClientMetadata -> ClientStates)
    - 0x3 = Dirty (Tip)
    - 0x0 = Nothing (invalid)
    - 0x1 = Branch (read-only access)
- Example
    - Writing to address 0x8000_1000 (tohost)
    - offset = 0b000_000
    - set_idx = 0b000_000
    - The core chooses which way to write to (way 1 (0010) is selected in rv64ui-p-simple)
    - The tag bits are 0x38_0001 (3 = upper 2 bits for metadata, dirty line) (raw tag is 0x8_0001)
    - This corresponds to writing the tag bits to bank 1 in address 0
- Oddly enough, writing to set_idx = 0 writes to the 0th entry of that way's tag array (as seen in the RTL waveform)
    - This is different from the register file where writes to register 1 actually writes to the 30th entry of the register file
    - But this is good news, it means no complex reversing logic is required

#### Data Array

- Each cache block is 64B, but the data array is organized as 32 banks of 512 addresses (9 address bits) x 8 bits data
    - Each access is 32B (256 bits) at a time
- Writes to the data cache come from a port with 12 bit addresses and 64 bits of data
- Here's the organization
    - The banks can be grouped into four 64 data bits banks (group 8 byte-wise banks together)
    - So now there is [Bank[3], Bank[2], Bank[1], Bank[0]], each Bank is 64-bits of data and has 512 addresses
    - Each of these banks corresponds to a single way
    - Within each way's bank, addresses 0-7 correspond to set 0, 8-15 to set 1, ..., 504-511 to set 63
