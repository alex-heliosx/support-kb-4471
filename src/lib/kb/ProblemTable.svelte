<script lang="ts">
  import reports from '#lib/data/reports-4471.json';
  import notes from '#lib/data/desk-notes.json';
  import { causeOf, uniqueById, type Report } from '#lib/kb/causes';
  import { tenantOf } from '#lib/kb/locales';

  // One row per problem in a merged ticket. Cause comes from causes.ts; reports the desk read as a
  // one-day move (desk-notes.json) are their own problem, not the cause their tenant points to.
  type Problem = { key: string; label: string; issue: string };
  let { ticket, problems }: { ticket: number; problems: Problem[] } = $props();

  const rows = $derived.by(() => {
    const raw = (reports as Report[]).filter((r) => r.mergedInto === ticket);
    const merged = uniqueById(raw);
    const problemOf = (r: Report) =>
      notes.oneDayMove.includes(r.id) ? 'one-day' : (causeOf(r, tenantOf(r)) ?? 'unplaced');
    const byKey = new Map<string, Report[]>();
    for (const r of merged) byKey.set(problemOf(r), [...(byKey.get(problemOf(r)) ?? []), r]);
    const known = problems.map((p) => ({ ...p, reports: byKey.get(p.key) ?? [] }));
    const rest = [...byKey.entries()]
      .filter(([k]) => !problems.some((p) => p.key === k))
      .map(([key, rs]) => ({ key, label: 'not placed', issue: '', reports: rs }));
    return { rows: [...known, ...rest], total: merged.length, rows427: raw.length };
  });

  const span = (rs: Report[]) => {
    const days = rs.map((r) => r.receivedAt.slice(0, 10)).sort();
    return days.length ? `${days[0]} → ${days[days.length - 1]}` : '—';
  };
</script>

<p class="sum">
  <strong>{rows.total}</strong> reports in #{ticket} ({rows.rows427 - rows.total} duplicate rows in the export dropped) ·
  <strong>{rows.rows.filter((r) => r.reports.length).length}</strong> problems
</p>

<table>
  <thead><tr><th>problem</th><th>reports</th><th>tenants</th><th>seen</th><th>ticket</th></tr></thead>
  <tbody>
    {#each rows.rows as p (p.key)}
      <tr>
        <td>{p.label} <code>{p.key}</code></td>
        <td>{p.reports.length}</td>
        <td>{new Set(p.reports.map((r) => r.tenant)).size}</td>
        <td>{span(p.reports)}</td>
        <td>{#if p.issue}<a href={p.issue}>{p.issue.replace(/^.*\/issues\//, '#')}</a>{:else}none{/if}</td>
      </tr>
    {/each}
  </tbody>
</table>

<style>
  .sum { font-size: 1.1rem; }
</style>
