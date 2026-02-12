<script lang="ts">
  import Papa from 'papaparse';
  
  export interface CsvEditorProps {
    class?: string;
    defaultFilename?: string;
  }

  let { class: className = '', defaultFilename = 'edited' }: CsvEditorProps = $props();

  type Row = Record<string, string>
  <!-- column names and rows -->
  let headers = $state<string[]>([]); 
  let rows = $state<Row[]>([]); 
  let fileNameBase = $state(defaultFilename); 
  let error = $state<string>(''); 

 <!--filter text -->
  let query = $state('');
  
  function parseCsvText(csvText: string) { 
    error = ''; 
    const result = Papa.parse<string[]>(csvText, { skipEmptyLines: true }); 
    if (result.errors?.length) { 
        error = result.errors[0]?.message || 'CSV parse error.'; 
        return; 
    } 
    
    const data = result.data ?? []; 
    if (!data.length) { 
        headers = []; 
        rows = []; 
        return; 
    } 
    headers = (data[0] ?? []).map((h) => String(h ?? '').trim()); 
    rows = (data.slice(1) ?? []).map((r) => { 
        const obj: Row = {}; 
        for (let i = 0; i < headers.length; i++) obj[headers[i]] = String(r?.[i] ?? ''); 
        return obj; 
    }); 
} 

function handleFileUpload(e: Event) { 
    const file = (e.currentTarget as HTMLInputElement).files?.[0]; 
    if (!file) return; 
    
    fileNameBase = file.name.replace(/\.[^/.]+$/, '') || defaultFilename; 
    
    const reader = new FileReader(); 
    reader.onload = () => parseCsvText(String(reader.result ?? '')); 
    reader.onerror = () => (error = 'Failed to read file.'); 
    reader.readAsText(file); } 
    
    function updateCell(rowIndex: number, col: string, value: string) { 
        rows = rows.map((r, i) => (i === rowIndex ? { ...r, [col]: value } : r)); 
    } 
    
    function addRow() { 
        if (headers.length === 0) { 
            headers = ['Column1', 'Column2']; 
        } 
        const newRow: Row = Object.fromEntries(headers.map((h) => [h, ''])); 
        rows = [...rows, newRow]; 
    } 
    
    function deleteRow(index: number) { 
        rows = rows.filter((_, i) => i !== index); 
    } 
    
    function downloadCsv() { 
        if (headers.length === 0) { 
            error = 'Nothing to download yet.'; 
            return; 
        } 
        
        const matrix: string[][] = [ 
            headers, 
            ...rows.map((r) => headers.map((h) => r[h] ?? ''))
        ]; 
        
        const csv = Papa.unparse(matrix); 
        const blob = new Blob([csv], { type: 'text/csv;charset=utf-8' }); 
        const url = URL.createObjectURL(blob); 
        
        const a = document.createElement('a'); 
        a.href = url; 
        a.download = `${fileNameBase}.csv`; 
        document.body.appendChild(a); 
        a.click(); 
        a.remove(); 
        
        URL.revokeObjectURL(url); 
    }

    const filteredRows = $derived.by(() => {
        const q = query.trim().toLowerCase();
        if (!q) return rows;

        return rows.filter((r) =>
            headerss.some((h) =>
            String(r[h] ?? '').toLowerCase().includes(q)
            )
        );
    });

</script> 
<div class="{className}"> 
    <input type="file" accept=".csv,text/csv" onchange={handleFileUpload} /> 
    <button type="button" onclick={addRow}>Add row</button> 
    <button type="button" onclick={downloadCsv} disabled={headers.length === 0}>Download CSV</button> 
    <input 
        placeholder="Filter..."
        bind:value={query}
    />
    
    {#if error} 
        <p>{error}</p> 
    {/if} 
    
    {#if headers.length > 0} 
        <table> 
            <thead> 
                <tr> 
                    {#each headers as h} 
                        <th>{h}</th> 
                    {/each} 
                    <th>Delete</th> 
                </tr> 
            </thead> 
            <tbody> 
                {#each filteredRows as r} 
                    {@const idx = rows.indexOf(r)}
                    <tr> 
                        {#each headers as h} 
                            <td> 
                                <input 
                                    value={r[h] ?? ''} 
                                    oninput={(e) => updateCell(idx, h, (e.currentTarget as HTMLInputElement).value)} 
                                /> 
                            </td> 
                        {/each} 
                        <td> 
                            <button type="button" onclick={() => deleteRow(i)}>Delete</button> 
                        </td> 
                    </tr> 
                {/each} 
            </tbody> 
        </table> 
    {/if} 
</div>