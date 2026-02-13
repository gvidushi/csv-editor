<script lang="ts">
  import Papa from 'papaparse';
  
  export interface CsvEditorProps {
    class?: string;
    defaultFilename?: string;
  }

  let { class: className = '', defaultFilename = 'edited' }: CsvEditorProps = $props();

  type Row = Record<string, string>
  let headers = $state<string[]>([]); 
  let rows = $state<Row[]>([]); 
  let fileNameBase = $state(defaultFilename); 
  let error: string = $state(''); 

  let query: string = $state('');
  
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
        query = '';
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
            headers.some((h) =>
            String(r[h] ?? '').toLowerCase().includes(q)
            )
        );
    });

</script> 
<div class="csv-wrap {className}"> 
    <div class="toolbar">
        <label class="file-btn">
            <input type="file" accept=".csv,text/csv" onchange={handleFileUpload} /> 
            Choose CSV
        </label>
        <button type="button" onclick={addRow}>Add row</button> 
        <button type="button" onclick={downloadCsv} disabled={headers.length === 0}>Download CSV</button> 
        <input 
            class="filter"
            placeholder="Filter..."
            bind:value={query}
            oninput={(e) => (query = (e.currentTarget as HTMLInputElement).value)}
        />
    </div>
    
    {#if error} 
        <p>{error}</p> 
    {/if} 
    
    {#if headers.length > 0} 
        <div class="table-wrap">
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
                                        title={r[h] ?? ''}
                                        oninput={(e) => updateCell(idx, h, (e.currentTarget as HTMLInputElement).value)} 
                                    /> 
                                </td> 
                            {/each} 
                            <td> 
                                <button type="button" onclick={() => deleteRow(idx)}>Delete</button> 
                            </td> 
                        </tr> 
                    {/each} 
                </tbody> 
            </table> 
        </div>
    {/if} 
</div>

<style>
  .csv-wrap {
    max-width: 1100px;
    margin: 96px auto 24px;
    padding: 16px;
  }

  .toolbar {
    display: flex;
    gap: 10px;
    align-items: center;
    flex-wrap: wrap;
    margin-bottom: 12px;
  }

  .toolbar button {
    padding: 8px 10px;
    border: 1px solid rgba(255,255,255,0.25);
    background: rgba(255,255,255,0.12);;
    color: white;
    border-radius: 8px;
    cursor: pointer;
  }

  .toolbar button:disabled {
    opacity: 0.5;
    cursor: not-allowed;
  }

  .filter {
    padding: 8px 10px;
    border: 1px solid #ddd;
    border-radius: 8px;
    min-width: 220px;
  }

  .error {
    color: #b00020;
    margin: 6px 0 12px;
  }

  .table-wrap {
    overflow-x: auto;
}

  table {
  min-width: 900px;
  width: 100%;
  border-collapse: collapse;
  border: 1px solid rgba(255,255,255,0.15);
  border-radius: 10px;
  overflow: hidden;
  background: rgba(0,0,0,0.18);
}

  thead th {
  text-align: left;
  font-weight: 600;
  color: rgba(255,255,255,0.9);
  background: rgba(255,255,255,0.08);
  border-bottom: 1px solid rgba(255,255,255,0.15);
  padding: 10px;
}

  tbody td {
  border-bottom: 1px solid rgba(255,255,255,0.10);
  padding: 8px 10px;
}

  tbody tr:last-child td {
  border-bottom: none;
}

  td input {
  width: 100%;
  padding: 6px 8px;
  border: none;             
  background: transparent; 
  color: white;
  outline: none;
  overflow-x: auto;
  white-space: nowrap;
}

  td input::placeholder {
  color: rgba(255,255,255,0.55);
}

  td input:focus {
  outline: none;
}

  td button {
  padding: 6px 10px;
  border: 1px solid rgba(255,255,255,0.25) !important;
  background: rgba(255,255,255,0.10) !important;
  color: white !important;
  border-radius: 8px;
  cursor: pointer;
}

  td button:hover {
  background: rgba(255,255,255,0.16) !important;
}


  .file-btn input {
  display: none;
}

  .file-btn {
  padding: 8px 12px;
  border-radius: 8px;
  border: 1px solid rgba(255,255,255,0.25);
  background: rgba(255,255,255,0.12);
  color: white;
  cursor: pointer;
  font-size: 14px;
}

  .file-btn:hover {
  background: rgba(255,255,255,0.18);
}


</style>