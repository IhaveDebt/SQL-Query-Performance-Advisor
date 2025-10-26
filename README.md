#!/usr/bin/env node
/**
 * SQL Query Performance Advisor (Node CLI)
 * Usage:
 *   node src/sql_advisor_cli.js "SELECT * FROM users WHERE email='a@b.com';"
 *
 * Heuristic, safe — does not execute queries.
 */

function parseSelect(q){
  q = q.trim();
  const m = q.match(/^select\s+(.+?)\s+from\s+([^\s;]+)(?:\s+where\s+(.+?))?(?:\s+order\s+by\s+(.+?))?/i);
  if (!m) return null;
  return { cols: m[1], table: m[2], where: m[3], order: m[4] };
}

function advise(q){
  const p = parseSelect(q);
  if (!p) return ["Could not parse query - try a simple SELECT statement."];
  const adv = [];
  if (/\*/.test(p.cols)) adv.push("Avoid `SELECT *` — project only required columns to reduce IO and network transfer.");
  if (p.where) {
    adv.push("Query has a WHERE clause — examine predicates and check for available indexes.");
    // suggest equality indexes
    const eqs = [...p.where.matchAll(/(\w+)\s*=\s*['"]?[\w\d@.\-]+['"]?/g)];
    for (const e of eqs) adv.push(`Candidate index: column \`${e[1]}\` for equality lookup.`);
    if (/like/i.test(p.where)) adv.push("`LIKE` may not use standard B-tree indexes; consider trigram or full-text index for pattern searches.");
  } else {
    adv.push("No WHERE clause — this will likely cause a full table scan; add predicates or limit.");
  }
  if (p.order) adv.push("ORDER BY present — consider covering indexes that match the ORDER BY to avoid sorting.");
  adv.push("Collect EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON) and compare actual vs estimated rows to find cardinality issues.");
  return adv;
}

// CLI
if (require.main === module) {
  const q = process.argv.slice(2).join(" ");
  if (!q) { console.log("Usage: node src/sql_advisor_cli.js \"SELECT ...\""); process.exit(1); }
  const out = advise(q);
  console.log("Advice:");
  out.forEach((a, i) => console.log(`${i+1}. ${a}`));
}

module.exports = { advise, parseSelect };
