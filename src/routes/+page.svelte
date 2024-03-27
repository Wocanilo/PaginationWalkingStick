<script lang="ts">
    import {Container, Form, Input, Label, FormGroup, FormText, Row, Col, Button} from '@sveltestrap/sveltestrap';
    import {writable} from "svelte/store";
    import EntryFlags from '../components/EntryFlags.svelte';

    // Parameters
    const targetAddress = writable("");

    // Addresses from inputs
    const pgdAddress = writable("");
    const pgdLocationAddress = writable("");
    const pgdEntry = writable("");

    // PUD
    const pudLocationAddress = writable("");
    const pudEntry = writable("");

    // PMD
    const pmdLocationAddress = writable("");
    const pmdEntry = writable("");

    // PT
    const ptLocationAddress = writable("");
    const ptEntry = writable("");

    // Number values
    let targetAddr: bigint = BigInt(0);
    let kernelAddr: bigint = BigInt(0);

    let pgdAddr: bigint = BigInt(0);
    let pudAddr: bigint = BigInt(0);
    let pmdAddr: bigint = BigInt(0);
    let ptAddr: bigint = BigInt(0);

    // Kernel Parameters
    const kernelOffset = writable("0xffff888000000000");

    // https://elixir.bootlin.com/linux/latest/source/arch/x86/include/asm/pgtable_64_types.h#L75
    const PGDIR_SHIFT: bigint = BigInt(39);
    const PTRS_PER_PGD: bigint = BigInt(512);

    /// 3rd Level (PUD)
    const PUD_SHIFT: bigint = BigInt(30);
    const PTRS_PER_PUD: bigint = BigInt(512);

    // 2nd level (PMD)
    const PMD_SHIFT: bigint = BigInt(21);
    const PTRS_PER_PMD: bigint = BigInt(512);

    // 1st Level (PT)
    // Not sure where this is defined in the kernel
    const PT_SHIFT: bigint = BigInt(12);
    const PTRS_PER_PT: bigint = BigInt(512);

    function loadExample(){
        targetAddress.set("0x404060");
        pgdAddress.set("0xffff88807c0a4000");
        pgdEntry.set("0x7c05f067");
        pudEntry.set("0x7c7c3067");
        pmdEntry.set("0x7c7e2067");
        ptEntry.set("0x8000000003305067");
    }

    function clear(){
        kernelOffset.set("0xffff888000000000");
        targetAddress.set("");
        pgdAddress.set("");
        pgdEntry.set("");
        pudEntry.set("");
        pmdEntry.set("");
        ptEntry.set("");
        pgdLocationAddress.set("");
        pudLocationAddress.set("");
        pmdLocationAddress.set("");
        ptLocationAddress.set("");
    }

    // https://elixir.bootlin.com/linux/latest/source/include/linux/pgtable.h#L88
    function pgd_index(address: bigint){
        return (address >> PGDIR_SHIFT) & (PTRS_PER_PGD - BigInt(1));
    }

    // https://elixir.bootlin.com/linux/latest/source/include/linux/pgtable.h#L136
    function pgd_offset_pgd(pgd: bigint, address: bigint){
        return pgd + pgd_index(address) * BigInt(8);
    }

    // https://elixir.bootlin.com/linux/latest/source/include/linux/pgtable.h#L79
    function pud_index(address: bigint){
        return (address >> PUD_SHIFT) & (PTRS_PER_PUD - BigInt(1));
    }

    // https://elixir.bootlin.com/linux/latest/source/include/linux/pgtable.h#L129
    function pud_offset(p4d: bigint, address: bigint){
        return p4d + pud_index(address) * BigInt(8);
    }

    // https://elixir.bootlin.com/linux/latest/source/include/linux/pgtable.h#L71
    function pmd_index(address: bigint){
        return (address >> PMD_SHIFT) & (PTRS_PER_PMD - BigInt(1));
    }

    // https://elixir.bootlin.com/linux/latest/source/include/linux/pgtable.h#L121
    function pmd_offset(pud: bigint, address: bigint){
        return pud + pmd_index(address) * BigInt(8);
    }

    // Not found on kernel source
    function pt_index(address: bigint){
        return (address >> PT_SHIFT) & (PTRS_PER_PT - BigInt(1));
    }

    function pt_offset(pt: bigint, address: bigint){
        return pt + pt_index(address) * BigInt(8);
    }

    function updateCalc(){
        if(pgdAddr != BigInt(0)) pgdLocationAddress.set("0x" + (kernelAddr + pgd_offset_pgd(pgdAddr, targetAddr)).toString(16));
        if(pudAddr != BigInt(0)) pudLocationAddress.set("0x" + (kernelAddr + pud_offset(pudAddr, targetAddr)).toString(16));
        if(pmdAddr != BigInt(0)) pmdLocationAddress.set("0x" + (kernelAddr + pmd_offset(pmdAddr, targetAddr)).toString(16));
        if(ptAddr != BigInt(0))  ptLocationAddress.set("0x" + (kernelAddr + pt_offset(ptAddr, targetAddr)).toString(16));
    }

    targetAddress.subscribe((value) => {
        try{
            targetAddr = BigInt(value, 16) & BigInt(~0xfff);
            updateCalc();
        }catch(e){}
    });

    kernelOffset.subscribe((value) => {
        try{
            kernelAddr = BigInt(value, 16);
            updateCalc();
        }catch(e){}
    });

    pgdAddress.subscribe((value) => {
        try{
            pgdAddr = BigInt(value, 16) & BigInt(~0xfff); // Clear flags
            pgdAddr = pgdAddr & ~kernelAddr; // Remove the kernel offset (if present)
            updateCalc();
        }catch(e){}
    });

    pgdEntry.subscribe((value) => {
        try{
            let pgdEntry2 = BigInt(value, 16);

            // Physical Address
            pudAddr = pgdEntry2 & BigInt(0x7fffffffff000); // Clear flags, only keep physical address
            
            updateCalc();
        }catch(e){}
    });

    pudEntry.subscribe((value) => {
        try{
            let pudEntry = BigInt(value, 16);

            // Physical Address
            pmdAddr = pudEntry & BigInt(0x7fffffffff000); // Clear flags, only keep physical address
            
            updateCalc();
        }catch(e){console.error(e)}
    });

    pmdEntry.subscribe((value) => {
        try{
            let pmdEntry = BigInt(value, 16);

            ptAddr = pmdEntry & BigInt(0x7fffffffff000); // Clear flags, only keep physical address

            updateCalc();
        }catch(e){console.error(e)}
    });

</script>
<Container class="py-5">
    <Row class="justify-content-center">
        <Col>
            <h1>
                Linux x86_64 Pagination Walking Stick
            </h1>
            <div class="form-text mb-3">For those times were we feel like going for a walk</div>
            <p>The goal of this page is to assist you during a manual Virtual Address to Physical Address translation. Given a Virtual Address and the Page Global Directory address of the process,
                this tool will calculate the different pointers you need to read to continue walking the page.</p>
            <p>Support for 2MB and 1GB pages is included. You will be notified when dealing with huge pages. Also, some small verifications of the pointers is included,
                in case you are leaking the incorrect value, however, this detection is minimal, so don't rely on it.
            </p>
                Code is based on the following resources:
                <ul>
                    <li><a href="https://zolutal.github.io/understanding-paging/" target="_blank">[1] Zolutal: Understanding x86_64 Paging</a></li>
                    <li><a href="https://elixir.bootlin.com/linux/latest/source/include/linux/pgtable.h#L88" target="_blank">[2] Kernel Source: linux/latest/source/include/linux/pgtable.h</a></li>
                    <li><a href="https://elixir.bootlin.com/linux/latest/source/arch/x86/include/asm/pgtable_types.h#L10" target="_blank">[3] Kernel Source: linux/latest/source/arch/x86/include/asm/pgtable_types.h</a></li>
                    <li><a href="https://elixir.bootlin.com/linux/latest/source/include/linux/sched.h#L748" target="_blank">[4] Kernel Source: linux/latest/source/include/linux/sched.h</a></li>
                </ul>                
            <h2 class="mb-3 pt-4">Parameters</h2>
            <Form>
                <FormGroup class="mb-3">
                    <Button color="warning" on:click={loadExample}>Load Example</Button>
                    <Button color="danger" on:click={clear}>Reset</Button>
                </FormGroup>
                <FormGroup class="mb-3">
                    <Label for="targetAddress">Target Process Virtual Memory Address:</Label>
                    <Input type="text" id="targetAddress" aria-describedby="targetAddressHelp" placeholder="0xdeadbeef1337" bind:value={$targetAddress}/>
                    <FormText id="targetAddressHelp">Virtual Address for which you want to find the associated page</FormText>
                </FormGroup>
                <FormGroup class="mx-5">
                    <Label for="level1Index">Level 1 - Page Table (PT) Index:</Label>
                    <Input type="text" disabled id="level1Index" aria-describedby="level1Help" value={`0x${pt_index(targetAddr).toString(16)}`}/>
                    <FormText id="level1Help">Index = (address &gt;&gt; 12) & 0x1ff</FormText>
                </FormGroup>
                <FormGroup class="mx-5">
                    <Label for="level2Index">Level 2 - Page Middle Directory (PMD) Index:</Label>
                    <Input type="text" disabled id="level2Index" aria-describedby="level2Help" value={`0x${pmd_index(targetAddr).toString(16)}`}/>
                    <FormText id="level2Help">Index = (address &gt;&gt; 21) & 0x1ff</FormText>
                </FormGroup>
                <FormGroup class="mx-5">
                    <Label for="level3Index">Level 3 - Page Upper Directory (PUD) Index:</Label>
                    <Input type="text" disabled id="level3Index" aria-describedby="level3Help" value={`0x${pud_index(targetAddr).toString(16)}`}/>
                    <FormText id="level3Help">Index = (address &gt;&gt; 30) & 0x1ff</FormText>
                </FormGroup>
                <FormGroup class="mx-5">
                    <Label for="level4Index">Level 4 - Page Global Directory (PGD) Index:</Label>
                    <Input type="text" disabled id="level4Index" aria-describedby="level4Help" value={`0x${pgd_index(targetAddr).toString(16)}`}/>
                    <FormText id="level4Help">Index = (address &gt;&gt; 39) & 0x1ff</FormText>
                </FormGroup>
                <FormGroup class="mb-3 pt-3">
                    <Label for="pgdAddress">Page Global Directory (PGD) Address:</Label>
                    <Input type="text" id="pgdAddress" aria-describedby="pgdHelp" bind:value={$pgdAddress}/>
                    <FormText id="pgdHelp">Located at task_struct-&gt;mm-&gt;pgd.<p>GDB: print ((struct task_struct*)Address)->mm->pgd</p></FormText>
                </FormGroup>
                <h3 class="mb-3">Advanced Parameters</h3>
                <FormGroup class="mb-3">
                    <Label for="kernelOffset">Kernel Direct Mapping of Physical Memory Address:</Label>
                    <Input type="text" id="kernelOffset" aria-describedby="kernelOffsetHelp" placeholder="0xdeadbeef" bind:value={$kernelOffset}/>
                    <FormText id="kernelOffsetHelp">Address in which Physical Memory is mapped in the Kernel <a target="_blank" href="https://www.kernel.org/doc/Documentation/x86/x86_64/mm.txt">https://www.kernel.org/doc/Documentation/x86/x86_64/mm.txt</a><p>Set to 0 if accesing physical memory directly</p></FormText>
                </FormGroup>
                <!-- Level 4 -->
                <h2 class="mb-2 pt-4">Level 4 - Page Global Directory (PGD) to Page Upper Directory (PUD)</h2>
                <FormGroup class="mb-3">
                    <Label for="pgdEntryAddress">PGD Entry At:</Label>
                    <Input type="text" id="pgdEntryAddress" aria-describedby="pgdEntryHelp" disabled bind:value={$pgdLocationAddress}/>
                    <FormText id="pgdEntryHelp">Address of the PGD in which the PUD address of the page is located</FormText>
                </FormGroup>
                <FormGroup class="mb-3">
                    <Label for="pgdAddress">Entry Value:</Label>
                    <Input type="text" id="pgdAddress" aria-describedby="pudEntryHelp" placeholder={`x/gx ${$pgdLocationAddress}`} bind:value={$pgdEntry}/>
                    <FormText id="pgdEntryHelp">Value of the PGD entry<p>{`x/gx ${$pgdLocationAddress}`}</p></FormText>
                </FormGroup>
                <EntryFlags addressInput={pgdEntry} addressType="PGD" virtualAddress={targetAddress} kernelOffset={kernelOffset}/>
                <!-- Level 3 -->
                <h2 class="mb-2 pt-4">Level 3 - Page Upper Directory (PUD) to Page Middle Directory (PMD)</h2>
                <FormGroup>
                    <Label for="pmdAddress">PUD Entry at:</Label>
                    <Input type="text" id="pmdAddress" disabled bind:value={$pudLocationAddress}/>
                    <FormText>Address of the PUD in which the PMD address of the page is located</FormText>
                </FormGroup>
                <FormGroup class="mb-3">
                    <Label for="pudAddress">Entry Value:</Label>
                    <Input type="text" id="pudAddress" aria-describedby="pudAddressHelp" placeholder={`x/gx ${$pudLocationAddress}`} bind:value={$pudEntry}/>
                    <FormText id="pudAddressHelp">Value of the PUD Entry<p>{`x/gx ${$pudLocationAddress}`}</p></FormText>
                </FormGroup>
                <EntryFlags addressInput={pudEntry} addressType="PUD" virtualAddress={targetAddress} kernelOffset={kernelOffset}/>
                <!-- Level 2 -->
                <h2 class="mb-2 pt-4">Level 2 - Page Middle Directory (PMD) to Page Table (PT)</h2>
                <FormGroup>
                    <Label for="ptLocated">PMD Entry at:</Label>
                    <Input type="text" id="ptLocated" disabled aria-describedby="ptLocatedhelp" bind:value={$pmdLocationAddress}/>
                    <FormText id="ptLocatedhelp">Address of the PMD where our entry is located</FormText>
                </FormGroup>
                <FormGroup class="mb-3">
                    <Label for="pmdEntry">Entry Value:</Label>
                    <Input type="text" id="pmdEntry" aria-describedby="pmdEntryHelp" bind:value={$pmdEntry} placeholder={`x/gx ${$pmdLocationAddress}`}/>
                    <FormText id="pmdEntryHelp">Value of the PMD Entry<p>{`x/gx ${$pmdLocationAddress}`}</p></FormText>
                </FormGroup>
                <EntryFlags addressInput={pmdEntry} addressType="PMD" virtualAddress={targetAddress} kernelOffset={kernelOffset}/>
                <!-- Level 1 -->
                <h2 class="pt-4">Level 1 - Page Table (PT) to Physical Address</h2>
                <FormGroup>
                    <Label for="physicalLocated">PT Entry at:</Label>
                    <Input type="text" id="physicalLocated" disabled aria-describedby="physicalLocatedhelp" bind:value={$ptLocationAddress}/>
                    <FormText id="physicalLocatedhelp">Address of the PT where our entry is located</FormText>
                </FormGroup>
                <FormGroup class="mb-3">
                    <Label for="pmdEntry">Entry Value:</Label>
                    <Input type="text" id="pmdEntry" aria-describedby="pmdEntryHelp" bind:value={$ptEntry} placeholder={`x/gx ${$ptLocationAddress}`}/>
                    <FormText id="pmdEntryHelp">Value of the PT Entry<p>{`x/gx ${$ptLocationAddress}`}</p></FormText>
                </FormGroup>
                <EntryFlags addressInput={ptEntry} addressType="PT" virtualAddress={targetAddress} kernelOffset={kernelOffset}/>
            </Form>
        </Col>
    </Row>
</Container>