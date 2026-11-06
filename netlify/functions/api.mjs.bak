import { getStore } from "@netlify/blobs";
const A = "https://api.apify.com/v2";
const ACTOR = process.env.APIFY_ACTOR || "datadoping~instagram-likes-scraper";
const json = (d, s = 200) => new Response(JSON.stringify(d), { status: s, headers: { "content-type": "application/json" } });
const clean = (u) => { const m = String(u).match(/instagram\.com\/(p|reel|reels|tv)\/([\w-]+)/i); return m ? `https://www.instagram.com/${m[1].toLowerCase() === "reels" ? "reel" : m[1].toLowerCase()}/${m[2]}/` : null; };

export default async (req) => {
  if (req.method !== "POST") return json({ error: "POST only" }, 405);
  if (!process.env.APP_PASSWORD || req.headers.get("x-app-password") !== process.env.APP_PASSWORD) return json({ error: "Password salah" }, 401);
  const token = process.env.APIFY_TOKEN;
  if (!token) return json({ error: "APIFY_TOKEN belum diisi di Netlify" }, 500);
  const store = getStore("spn");
  let jobs = (await store.get("jobs", { type: "json" })) || [];
  let likers = (await store.get("likers", { type: "json" })) || [];
  const body = await req.json().catch(() => ({}));
  const now = () => new Date().toISOString();

  if (body.action === "add") {
    const urls = [...new Set((body.urls || []).map(clean).filter(Boolean))].filter((u) => !jobs.some((j) => j.url === u && j.status !== "FAILED")).slice(0, 10);
    for (const url of urls) {
      const r = await fetch(`${A}/acts/${ACTOR}/runs?token=${token}`, { method: "POST", headers: { "content-type": "application/json" }, body: JSON.stringify({ posts: [url], max_count: Math.min(Number(body.max) || 1000, 5000) }) });
      const d = (await r.json().catch(() => ({}))).data;
      jobs.unshift({ id: crypto.randomUUID(), url, runId: d?.id, datasetId: d?.defaultDatasetId, status: d ? "RUNNING" : "FAILED", added: now(), count: 0 });
    }
  }
  if (body.action === "delete") {
    jobs = jobs.filter((j) => j.id !== body.id);
    likers = likers.filter((l) => l.job !== body.id);
  }
  for (const j of jobs.filter((j) => j.status === "RUNNING")) {
    const d = (await (await fetch(`${A}/actor-runs/${j.runId}?token=${token}`)).json()).data;
    if (d?.status === "SUCCEEDED") {
      const items = await (await fetch(`${A}/datasets/${j.datasetId}/items?token=${token}&clean=true&format=json`)).json();
      const seen = new Set(likers.filter((l) => l.job === j.id).map((l) => l.u));
      for (const it of items) if (it.username && !seen.has(it.username)) { seen.add(it.username); likers.push({ u: it.username, n: it.full_name || "", post: j.url, job: j.id, at: now() }); }
      j.status = "SUCCEEDED"; j.count = seen.size;
    } else if (["FAILED", "ABORTED", "TIMED-OUT"].includes(d?.status)) j.status = "FAILED";
  }
  await store.setJSON("jobs", jobs);
  await store.setJSON("likers", likers);
  return json({ jobs, likers });
};
export const config = { path: "/api" };
