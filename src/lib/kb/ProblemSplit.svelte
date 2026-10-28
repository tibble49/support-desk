<script lang="ts">
  import reports from '#lib/data/reports-4471.json';
  import notes from '#lib/data/desk-notes.json';
  import { causeOf, uniqueById, type Report } from '#lib/kb/causes';
  import { tenantOf } from '#lib/kb/locales';

  let { ticket }: { ticket: number } = $props();

  const rows = reports as Report[];
  const merged = $derived(uniqueById(rows).filter((r) => r.mergedInto === ticket));
  const repeats = $derived(rows.filter((r) => r.mergedInto === ticket).length - merged.length);

  // Counted the desk's way: where the tenant files each report. The one-day reports are also a problem
  // of their own, so that row overlaps the one they are filed under.
  const problem = (r: Report) => (notes.oneDayMove.includes(r.id) ? ['one-day', causeOf(r, tenantOf(r))] : [causeOf(r, tenantOf(r))]);

  const of = (p: string) => merged.filter((r) => problem(r).includes(p as never));
  const count = (p: string) => of(p).length;
  const tagged = (p: string, tag: string) => of(p).filter((r) => r.meta.intakeTag === tag).length;
  const tenants = (p: string) => new Set(of(p).map((r) => r.tenant)).size;

  const table = $derived(
    [
      { p: 'parser', what: 'shared parser reads dates month first' },
      { p: 'hint', what: 'booking-v3 form hints MM/DD/YYYY to day-first tenants' },
      { p: 'one-day', what: 'bookings land exactly one day off' },
    ].map((x) => ({ ...x, n: count(x.p), tenants: tenants(x.p), wrongTag: tagged(x.p, x.p === 'hint' ? 'parser' : 'hint') })),
  );
</script>

<p class="sum">
  <strong>{merged.length}</strong> unique reports merged into #{ticket} ({repeats} repeats in the export removed) ·
  <strong>{table.length}</strong> problems
</p>

<table>
  <thead><tr><th>problem</th><th>reports</th><th>tenants</th><th>tagged wrong by intake</th></tr></thead>
  <tbody>
    {#each table as t (t.p)}
      <tr><td><code>{t.p}</code> · {t.what}</td><td>{t.n}</td><td>{t.tenants}</td><td>{t.wrongTag}</td></tr>
    {/each}
  </tbody>
</table>
