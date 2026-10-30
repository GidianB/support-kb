<script lang="ts">
  import reports from '#lib/data/reports-4471.json';
  import notes from '#lib/data/desk-notes.json';
  import { causeOf, uniqueById, type Report } from '#lib/kb/causes';
  import { tenantOf } from '#lib/kb/locales';

  // `issue` is the problem's own ticket in `repo` (GitHub owner/name).
  type Problem = { key: string; what: string; issue: number };
  let { ticket, repo, problems }: { ticket: number; repo: string; problems: Problem[] } = $props();

  const merged = $derived(uniqueById(reports as Report[]).filter((r) => r.mergedInto === ticket));
  // A one-day move (desk-notes.json) is its own bug, whatever cause its tenant would give it.
  const problemOf = (r: Report) => (notes.oneDayMove.includes(r.id) ? 'one-day' : (causeOf(r, tenantOf(r)) ?? 'unplaced'));
  const rows = $derived(
    problems.map((p) => {
      const rs = merged.filter((r) => problemOf(r) === p.key);
      const days = rs.map((r) => r.receivedAt.slice(0, 10)).sort();
      return { ...p, reports: rs.length, tenants: new Set(rs.map((r) => r.tenant)).size, firstSeen: days[0], lastSeen: days.at(-1) };
    }),
  );
  const unplaced = $derived(merged.length - rows.reduce((n, r) => n + r.reports, 0));
</script>

<p class="sum">
  <strong>{merged.length}</strong> reports merged into #{ticket} ·
  <strong>{rows.filter((r) => r.reports).length}</strong> problems, one ticket each
</p>

<table>
  <thead>
    <tr><th>problem</th><th>reports</th><th>tenants</th><th>seen</th><th>ticket</th></tr>
  </thead>
  <tbody>
    {#each rows as r (r.key)}
      <tr>
        <td>{r.what} <code>{r.key}</code></td>
        <td>{r.reports}</td>
        <td>{r.tenants}</td>
        <td>{r.reports ? `${r.firstSeen} → ${r.lastSeen}` : '—'}</td>
        <td><a href="https://github.com/{repo}/issues/{r.issue}">{repo}#{r.issue}</a></td>
      </tr>
    {/each}
  </tbody>
</table>

{#if unplaced}
  <p class="note">{unplaced} reports fit none of these problems. Read them before you merge them anywhere.</p>
{/if}

<style>
  .sum { font-size: 1.1rem; }
  .note { color: var(--warn); font-size: 0.85em; }
</style>
