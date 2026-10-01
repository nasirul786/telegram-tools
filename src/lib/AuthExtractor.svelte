<script lang="ts">
  import { addToast } from './components/toast.svelte.js';
  import {
    detectFormat,
    decodeSession,
    bytesToHex
  } from './engine';
  import FormatBadge from './components/FormatBadge.svelte';
  import { fade } from 'svelte/transition';

  let inputSession = $state<string>('');
  
  // Parse multiple sessions
  let parsedSessions = $derived(
    inputSession
      .split(/[\n,]+/)
      .map(s => s.trim())
      .filter(s => s.length > 0)
  );

  let detections = $derived(parsedSessions.map(s => detectFormat(s)));

  let decodedDataList = $derived.by(() => {
    return parsedSessions.map((s, i) => {
      const fmt = detections[i]?.format;
      if (!fmt) return null;
      try {
        return decodeSession(s, fmt);
      } catch {
        return null;
      }
    });
  });

  let hasValidSession = $derived(decodedDataList.some(d => d !== null));
  let firstDetection = $derived(detections[0] || null);
  
  let generatedResult = $state<string | null>(null);

  async function handleExtract() {
    if (!hasValidSession) {
      addToast('Cannot extract: no valid sessions found.', 'error');
      return;
    }

    try {
      const results: string[] = [];
      let successCount = 0;
      let failCount = 0;
      
      for (let i = 0; i < parsedSessions.length; i++) {
        const decoded = decodedDataList[i];
        if (!decoded) {
          failCount++;
          continue;
        }
        
        try {
          const hexKey = bytesToHex(decoded.authKey);
          results.push(`${hexKey}:${decoded.dcId}`);
          successCount++;
        } catch (e) {
          failCount++;
        }
      }
      
      generatedResult = results.join('\n');
      if (failCount > 0) {
        addToast(`Extracted ${successCount} auth key(s), failed ${failCount}.`, 'warning');
      } else {
        addToast(`Extracted ${successCount} auth key(s) successfully.`, 'success');
      }
    } catch (err: any) {
      addToast(`Extraction failed: ${err.message}`, 'error');
    }
  }

  async function handlePaste() {
    try {
      inputSession = await navigator.clipboard.readText();
      addToast('Pasted from clipboard', 'info');
    } catch {
      addToast('Failed to read clipboard', 'error');
    }
  }

  async function handleCopy() {
    if (!generatedResult) return;
    try {
      await navigator.clipboard.writeText(generatedResult);
      addToast('Copied to clipboard', 'success');
    } catch {
      addToast('Failed to copy', 'error');
    }
  }
</script>

<div class="grid grid-cols-1 lg:grid-cols-2 gap-6 items-start">
  
  <!-- Input Section -->
  <div class="flex flex-col gap-5">
    <!-- Input Area -->
    <div class="flex flex-col gap-2">
      <div class="flex items-center justify-between">
        <label for="ext-input" class="text-xs font-medium text-surface-400 uppercase tracking-wider">Input Session(s)</label>
        <button class="text-xs text-primary-400 hover:text-primary-300 font-medium transition-colors flex items-center gap-1 bg-primary-400/10 px-2 py-1 rounded-md" onclick={handlePaste}>
          <svg class="w-3.5 h-3.5" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M9 5H7a2 2 0 00-2 2v12a2 2 0 002 2h10a2 2 0 002-2V7a2 2 0 00-2-2h-2M9 5a2 2 0 002 2h2a2 2 0 002-2M9 5a2 2 0 012-2h2a2 2 0 012 2"></path></svg>
          Paste
        </button>
      </div>
      <div class="relative">
        <textarea
          id="ext-input"
          bind:value={inputSession}
          placeholder="Paste one or multiple sessions (1 per line or comma separated)..."
          class="w-full h-40 bg-surface-950 border border-surface-800 text-surface-200 text-sm font-mono p-4 rounded-xl resize-none focus:outline-none focus:ring-2 focus:ring-warning-500/40 focus:border-warning-500/40 transition-all"
          spellcheck="false"
        ></textarea>
        
        {#if inputSession.trim()}
          <div class="absolute bottom-3 right-3 flex items-center gap-2" transition:fade>
            {#if firstDetection}
              <div class="bg-surface-900 border border-surface-700 rounded-lg px-2 py-1.5 flex items-center gap-2">
                <span class="text-xs text-surface-400">Auto:</span>
                <FormatBadge format={firstDetection.format} />
                {#if parsedSessions.length > 1}
                  <span class="text-xs text-surface-400 border-l border-surface-700 pl-2 ml-1">+{parsedSessions.length - 1} more</span>
                {/if}
              </div>
            {/if}
          </div>
        {/if}
      </div>
    </div>

    <!-- Actions -->
    <div class="flex gap-2 pt-1">
      <button
        class="flex-1 bg-warning-600 hover:bg-warning-500 text-white font-medium py-2.5 px-4 rounded-lg transition-colors active:scale-[0.98] flex justify-center items-center gap-2 disabled:opacity-50 disabled:cursor-not-allowed"
        onclick={handleExtract}
        disabled={!inputSession.trim() || !hasValidSession}
      >
        Extract {#if parsedSessions.length > 1}({parsedSessions.length}){/if}
        <svg class="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M13 10V3L4 14h7v7l9-11h-7z"></path></svg>
      </button>
    </div>
  </div>

  <!-- Output Section -->
  <div class="flex flex-col gap-4">
    {#if generatedResult}
      <div class="flex flex-col gap-3" transition:fade>
        <div class="flex items-center justify-between">
          <span class="text-xs font-semibold text-surface-400 uppercase tracking-wider flex items-center gap-1.5">
            <svg class="w-3.5 h-3.5 text-success-500" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M5 13l4 4L19 7"></path></svg>
            Result (authkey:dcid)
          </span>
          <button class="text-xs text-primary-400 hover:text-primary-300 font-medium transition-colors flex items-center gap-1 bg-primary-400/10 px-2 py-1 rounded-md" onclick={handleCopy}>
            <svg class="w-3.5 h-3.5" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M8 5H6a2 2 0 00-2 2v12a2 2 0 002 2h10a2 2 0 002-2v-1M8 5a2 2 0 002 2h2a2 2 0 002-2M8 5a2 2 0 012-2h2a2 2 0 012 2m0 0h2a2 2 0 012 2v3m2 4H10m0 0l3-3m-3 3l3 3"></path></svg>
            Copy All
          </button>
        </div>
        
        <textarea
          readonly
          value={generatedResult}
          class="w-full h-64 bg-surface-950 border border-surface-800 text-surface-200 text-sm font-mono p-4 rounded-xl resize-none focus:outline-none focus:ring-2 focus:ring-warning-500/40 focus:border-warning-500/40 transition-all selection:bg-warning-500/30"
        ></textarea>
      </div>
    {:else if !hasValidSession && !inputSession.trim()}
      <!-- Empty State -->
      <div class="h-full min-h-[400px] flex flex-col items-center justify-center p-8 text-center border border-dashed border-surface-800 rounded-xl">
        <div class="w-12 h-12 rounded-lg bg-surface-900 flex items-center justify-center mb-4 text-surface-600">
          <svg class="w-6 h-6" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="1.5" d="M15 7a2 2 0 012 2m4 0a6 6 0 01-7.743 5.743L11 17H9v2H7v2H4v-3l8.44-8.44A6 6 0 0115 7zm-4 0h.01"></path></svg>
        </div>
        <p class="text-surface-400 text-sm font-medium mb-1">Ready to Extract</p>
        <p class="text-xs text-surface-600 max-w-xs">Paste your session string(s) to extract auth keys and DC IDs.</p>
      </div>
    {/if}
  </div>
</div>
