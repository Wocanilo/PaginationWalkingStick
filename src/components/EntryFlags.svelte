<script lang="ts">
    import {Row, Tooltip, Alert} from '@sveltestrap/sveltestrap';
	import type { Writable } from 'svelte/store';

    export let virtualAddress: Writable<String>;
    export let addressInput: Writable<String>;
    export let kernelOffset: Writable<String>;
    export let addressType: String;

    // Local vars
    let present: string;
    let readwrite: string;
    let usersupervisor: string;
    let writethrough: string;
    let cachedisabled: string;
    let accessed: string;
    let dirty: string;
    let global: string;
    let pageattributepage: string;
    let nx: string;
    let physical: bigint;
    let physicalAddressStr: string;
    let memoryprotection: string;
    let hugepage: string;

    let resultAddress: bigint = BigInt(0);

    // Flags offsets
    // https://elixir.bootlin.com/linux/latest/source/arch/x86/include/asm/pgtable_types.h#L10
    const BIT_PRESENT = BigInt(1); // Is present
    const BIT_WRITABLE = BigInt(2); // writeable
    const BIT_USER = BigInt(1) << BigInt(2); // userspace addressable
    const BIT_PWT = BigInt(1) << BigInt(3); // page write through
    const BIT_PCD = BigInt(1) << BigInt(4); // page cache disabled
    const BIT_ACCESSED = BigInt(1) << BigInt(5); // was accessed (raised by CPU)
    const BIT_DIRTY = BigInt(1) << BigInt(6); // was written to (raised by CPU)
    const BIT_PSE = BigInt(1) << BigInt(7); // 4 MB (or 2MB) page
    const BIT_PAT = BigInt(1) << BigInt(7); // on 4KB pages
    const BIT_GLOBAL = BigInt(1) << BigInt(8); // Global TLB entry PPro+
    const BIT_PAT_LARGE = BigInt(1) << BigInt(12); // On 2MB or 1GB pages
    const BIT_PKEY_BIT0 = BigInt(1) << BigInt(59); // Protection Keys, bit 1/4
    const BIT_PKEY_BIT1 = BigInt(1) << BigInt(60); // Protection Keys, bit 2/4
    const BIT_PKEY_BIT2 = BigInt(1) << BigInt(61); // Protection Keys, bit 3/4
    const BIT_PKEY_BIT3 = BigInt(1) << BigInt(62); // Protection Keys, bit 4/4
    const BIT_NX = BigInt(1) << BigInt(63); // No execute: only valid after cpuid check
    
    addressInput.subscribe((value) => {
        try{
            let address = BigInt(value, 16);
            let virtAddress = BigInt($virtualAddress, 16);

            if(address == BigInt(0)){
                // Invalid address, reset attributes
                present = "Unknown";
                readwrite = "Unknown";
                usersupervisor = "Unknown";
                writethrough = "Unknown";
                cachedisabled = "Unknown";
                accessed = "Unknown";
                dirty = "Unknown";
                global = "Unknown";
                pageattributepage = "Unknown";
                nx = "Unknown";
                physicalAddressStr = "Unknown";
                memoryprotection = "Unknown";
                hugepage = "Unknown";
                resultAddress = BigInt(0);
                return;
            }
            // Calculate flags
            if((address & BIT_PRESENT) == BIT_PRESENT){
                present = "True";
            }else{
                present = "False";
            }

            if((address & BIT_WRITABLE) == BIT_WRITABLE){
                readwrite = "Writable";
            }else{
                readwrite = "Read Only";
            }

            if((address & BIT_USER) == BIT_USER){
                usersupervisor = "User";
            }else{
                usersupervisor = "Supervisor";
            }

            if((address & BIT_PWT) == BIT_PWT){
                writethrough = "True";
            }else{
                writethrough = "False";
            }

            if((address & BIT_PCD) == BIT_PCD){
                cachedisabled = "True";
            }else{
                cachedisabled = "False";
            }

            if((address & BIT_ACCESSED) == BIT_ACCESSED){
                accessed = "True";
            }else{
                accessed = "False";
            }

            if((address & BIT_DIRTY) == BIT_DIRTY){
                dirty = "True";
            }else{
                dirty = "False";
            }

            if((address & BIT_GLOBAL) == BIT_GLOBAL){
                global = "True";
            }else{
                global = "False";
            }

            if((address & BIT_PAT_LARGE) == BIT_PAT_LARGE){
                pageattributepage = "True";
            }else{
                pageattributepage = "False";
            }

            if((address & BIT_NX) == BIT_NX){
                nx = "True";
            }else{
                nx = "False";
            }

            memoryprotection = ((address & (BIT_PKEY_BIT0 | BIT_PKEY_BIT1 | BIT_PKEY_BIT2 | BIT_PKEY_BIT3)) >> BigInt(59)).toString(2).padStart(4, "0");
            
            // Huge Page
            if((address & BIT_PSE) == BIT_PSE){
                hugepage = "True";
                if(addressType == "PMD"){
                    physical = (address & BigInt(0x7ffffffe00000));
                    // Is a 2MB huge page. The offset into the big table is VirtualAddress & ((1 << 21) - 1)
                    resultAddress =  physical + (virtAddress &  BigInt(0x1fffff));
                }
                else if(addressType == "PUD"){
                    // Is a 1GB huge page. The offset into the big table is VirtualAddress & ((1 << 30) - 1)
                    physical = (address & BigInt(0x7ffffc0000000));
                    resultAddress =  physical + (virtAddress &  BigInt(0x3fffffff));
                }
            }else{
                hugepage = "False";
                physical = address & BigInt(0x7fffffffff000); // Clear flags, only keep physical address
            }

            if(addressType == "PT") resultAddress = physical + (virtAddress & BigInt(0xfff)); // Last bits represent offset within the page 

            physicalAddressStr = "0x" + physical.toString(16); 
        }catch(e){

        }
    });

</script>

<h4>Decoded Address</h4>
<Row>
    {#if present == "False"}
        <Alert color="danger" class="pt-2 pb-2 mt-2">
            <h4>Invalid Entry</h4>
            The <b>Present</b> flag is not set. This entry won't be used for address translation
        </Alert>
    {/if}
    <div id="present-{addressType}">
        <b>Present: </b>{present}
        <Tooltip target="present-{addressType}" placement="left">
            If this bit is unset then the processor will not use this entry for address translation and none of the other bits will apply.
        </Tooltip>
    </div>
    <div id="readwrite-{addressType}">
        <b>Read/Write: </b>{readwrite}
        <Tooltip target="readwrite-{addressType}" placement="left">
            If this bit is unset, descendant pages cannot be written to.
        </Tooltip>
    </div>
    <div id="usersupervisor-{addressType}">
        <b>User/Supervisor: </b>{usersupervisor}
        <Tooltip target="usersupervisor-{addressType}" placement="left">
            If this bit is unset, descendant pages cannot be accessed unless in supervisor mode.
        </Tooltip>
    </div>
    <div id="pagewrite-{addressType}">
        <b>Page Write Through: </b>{writethrough}
        <Tooltip target="pagewrite-{addressType}" placement="left">
            If enabled, writes to descendant pages should immediately write to RAM rather than buffering writes to CPU cache before eventually updating RAM.
        </Tooltip>
    </div>
    <div id="pagecache-{addressType}">
        <b>Page Cache Disabled: </b>{cachedisabled}
        <Tooltip target="pagecache-{addressType}" placement="left">
            If enabled, descendant pages should not enter the CPU's cache hierarchy, sometimes also called the 'Uncacheable' (UC) bit.
        </Tooltip>
    </div>
    <div id="accessed-{addressType}">
        <b>Accessed: </b>{accessed}
        <Tooltip target="accessed-{addressType}" placement="left">
            If any pages referred to by this entry or its descendants, this bit will be set by the MMU, and can be cleared by the OS.
        </Tooltip>
    </div>
    {#if addressType != "PGD" && addressType != "PT"}
        <div id="pagesize-{addressType}">
            <b>Page Size: </b> {#if hugepage == "True"} <b>True</b> {:else} {hugepage} {/if}
            <Tooltip target="pagesize-{addressType}" placement="left">
                If set, the current entry holds the physical address of the page we are looking for
            </Tooltip>
        </div>
    {/if}
    <div id="physicaladdress-{addressType}">
        <b>PT Physical Address: </b>{physicalAddressStr}
        <Tooltip target="physicaladdress-{addressType}" placement="left">
            The physical address of the PT associated with this entry.
        </Tooltip>
    </div>
    <div id="nx-{addressType}">
        <b>NX: </b>{nx}
        <Tooltip target="nx-{addressType}" placement="left">
            If this bit is set, no memory mapping that is a descendant of this entry will be executable.
        </Tooltip>
    </div>
    {#if (hugepage == "True" || addressType == "PT") && addressType != "PGD"}
        <div id="dirty-{addressType}">
            <b>Dirty: </b>{dirty}
            <Tooltip target="dirty-{addressType}" placement="left">
                This bit is similar to the accessed bit, it gets set by the MMU if this page was written to and must be reset by the OS.
            </Tooltip>
        </div>
        <div id="global-{addressType}">
            <b>Global: </b>{global}
            <Tooltip target="global-{addressType}" placement="left">
                This has to do with how the TLB (Translation Lookaside Buffer, the MMU's cache for virtual to physical address translations) caches the translation for th page, this bit being set means the page will not be flushed from the TLB on context switch, this is commonly enabled on Kernel pages to reduce TLB misses.
            </Tooltip>
        </div>
        <div id="pageattribute-{addressType}">
            <b>Page Attribute Table: </b>{pageattributepage}
            <Tooltip target="pageattribute-{addressType}" placement="left">
                If set, the MMU should consult the Page Attribute Table MSR when determining whether the 'Memory Type' of the page, e.g. whether this page is 'Uncacheable', 'Write Through', or one of a few other memory types.
            </Tooltip>
        </div>
        <div id="memoryprotection-{addressType}">
            <b>Memory Protection Key: </b>{memoryprotection}
            <Tooltip target="memoryprotection-{addressType}" placement="left">
                This is an x86_64 extension that allows assigning a 4-bit keys to pages which can be used to configure memory permissions for all pages with that key.
            </Tooltip>
        </div>
    {/if}
    {#if hugepage == "True" && addressType == "PMD"}
        <Alert class="mt-4" color="success">
            <h3>Huge Page</h3>
            <p>The entry of the PMD has the <b>Page Size</b> flag enabled. This means that the target virtual address refers to a <b>2MB Huge Page</b> whose physical address is
                in this PMD entry.</p>

            The Virtual Address <b>{$virtualAddress}</b> is located in physical memory at <b>{`0x${resultAddress.toString(16)}`}</b> (Accessible at <b>{`0x${(resultAddress + BigInt($kernelOffset, 16)).toString(16)}`}</b>)
        </Alert>
    {/if}
    {#if hugepage == "True" && addressType == "PUD"}
        <Alert class="mt-4" color="success">
            <h3>Huge Page</h3>
            <p>The entry of the PUD has the <b>Page Size</b> flag enabled. This means that the target virtual address refers to a <b>1GB Huge Page</b> whose physical address is
                in this PUD entry.</p>

            The Virtual Address <b>{$virtualAddress}</b> is located in physical memory at <b>{`0x${resultAddress.toString(16)}`}</b> (Accessible at <b>{`0x${(resultAddress + BigInt($kernelOffset, 16)).toString(16)}`}</b>)
        </Alert>
    {/if}
    {#if addressType == "PT" && resultAddress != BigInt(0)}
    <Alert class="mt-4" color="success">
        The Virtual Address <b>{$virtualAddress}</b> is located in physical memory at <b>{`0x${resultAddress.toString(16)}`}</b> (Accessible at <b>{`0x${(resultAddress + BigInt($kernelOffset, 16)).toString(16)}`}</b>)
    </Alert>
    {/if}
</Row>