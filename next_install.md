npx create-next-app@12.3.3 todo-app --typescript

https://zgbnpkhciscoxkpqdesn.supabase.co

eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJzdXBhYmFzZSIsInJlZiI6InpnYm5wa2hjaXNjb3hrcHFkZXNuIiwicm9sZSI6ImFub24iLCJpYXQiOjE3NjQ1NTgyOTAsImV4cCI6MjA4MDEzNDI5MH0.1aAeak5leG6uxyMEO7_W_dG94wMf_grxXxUWqfF6iF8

npm install react@18 react-dom@18

npm install @heroicons/react@1.0.6 @supabase/supabase-js@1.33.3 @tanstack/react-query@4 zustand@3.7.1

npm install -D tailwindcss postcss autoprefixer

npm install -D prettier prettier-plugin-tailwindcss

npx tailwindcss init -p

### supabase functionsの作成方法

### 新しく関数のファイルの作成

C:\Users\masaki_yoneno\supabase\functionsにファイルが作成される

```
supabase functions new fred
```

index.tsを書き換える

```java
  import { serve } from "https://deno.land/std/http/server.ts"

const API_KEY = "9d56bbc6692463ab71bc51945c73c0ca"  // ← 自分のFRED APIキーに置き換えてね

// --- 期間計算 ---
function getStartDateByRange(range?: string): string {
  const today = new Date()
  switch (range) {
    case "1m": today.setMonth(today.getMonth() - 1); break
    case "3m": today.setMonth(today.getMonth() - 3); break
    case "6m": today.setMonth(today.getMonth() - 6); break
    case "1y": today.setFullYear(today.getFullYear() - 1); break
    case "3y": today.setFullYear(today.getFullYear() - 3); break
    case "5y": today.setFullYear(today.getFullYear() - 5); break
    case "10y": today.setFullYear(today.getFullYear() - 10); break
    default: today.setFullYear(today.getFullYear() - 10)
  }
  return today.toISOString().slice(0, 10)
}

interface Observation {
  date: string
  value: number | null
}

// --- FRED API fetch ---
async function fetchSeries(seriesId: string, startDate: string): Promise<Observation[]> {
  const url = `https://api.stlouisfed.org/fred/series/observations?series_id=${seriesId}&api_key=${API_KEY}&file_type=json&observation_start=${startDate}`
  const res = await fetch(url)
  const data = await res.json()

  if (!data.observations) return []

  return data.observations
    .map((v: any) => ({ date: v.date, value: v.value === "." ? null : Number(v.value) }))
    .sort((a, b) => a.date.localeCompare(b.date))
}

// --- データ揃える ---
function unifySeries(fx: Observation[], nikkei: Observation[], djia: Observation[]) {
  const latestDate = [fx, nikkei, djia].map(s => s[s.length - 1]?.date).sort().reverse()[0]
  const startDate = fx[0].date
  const labels: string[] = []
  const d = new Date(startDate)
  const lastDate = new Date(latestDate)

  while (d <= lastDate) {
    labels.push(d.toISOString().slice(0, 10))
    d.setDate(d.getDate() + 1)
  }

  const fill = (series: Observation[]): (number | null)[] => {
    const map: Record<string, number | null> = {}
    series.forEach(v => (map[v.date] = v.value))
    let lastVal: number | null = null
    return labels.map(date => {
      if (map[date] != null) { lastVal = map[date]; return map[date] }
      return lastVal
    })
  }

  return {
    labels,
    fx: fill(fx),
    nikkei: fill(nikkei),
    djia: fill(djia)
  }
}

// --- Edge Function 本体 ---
serve(async (req) => {
  // --- CORS プリフライト対応 ---
  if (req.method === "OPTIONS") {
    return new Response("ok", {
      headers: {
        "Access-Control-Allow-Origin": "*",
        "Access-Control-Allow-Methods": "GET, OPTIONS",
        "Access-Control-Allow-Headers": "*",
      },
    })
  }

  try {
    const url = new URL(req.url)
    const range = url.searchParams.get("range") ?? "10y"
    const startDate = getStartDateByRange(range)

    // --- データ取得 ---
    const [fx, nikkei, djia] = await Promise.all([
      fetchSeries("DEXJPUS", startDate),
      fetchSeries("NIKKEI225", startDate),
      fetchSeries("DJIA", startDate)
    ])

    const unified = unifySeries(fx, nikkei, djia)

    // --- JSONレスポンス ---
    return new Response(JSON.stringify({
      labels: unified.labels,
      datasets: [
        { label: "為替（USD/JPY）", data: unified.fx, borderColor: "blue", borderWidth: 1, pointRadius: 1, pointHoverRadius: 3, lineTension: 0.2 },
        { label: "日経平均", data: unified.nikkei, borderColor: "rgba(163, 15, 114, 1)", borderWidth: 1, pointRadius: 1, pointHoverRadius: 3, lineTension: 0.2 },
        { label: "ダウ平均", data: unified.djia, borderColor: "cornflowerblue", borderWidth: 1, pointRadius: 1, pointHoverRadius: 3, lineTension: 0.2 }
      ]
    }), {
      headers: {
        "Content-Type": "application/json",
        "Access-Control-Allow-Origin": "*",
        "Access-Control-Allow-Headers": "*",
      }
    })

  } catch (err) {
    return new Response(JSON.stringify({ error: String(err) }), {
      status: 500,
      headers: {
        "Content-Type": "application/json",
        "Access-Control-Allow-Origin": "*",
      }
    })
  }
})
```

### Edge Function にデプロイする

```
supabase functions deploy fred --no-verify-jwt
```

###　カールで確認する

```
Invoke-WebRequest `
  -Uri "https://zgbnpkhciscoxkpqdesn.supabase.co/functions/v1/fred" `
  -Headers @{
    "Authorization" = "Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJzdXBhYmFzZSIsInJlZiI6InpnYm5wa2hjaXNjb3hrcHFkZXNuIiwicm9sZSI6ImFub24iLCJpYXQiOjE3NjQ1NTgyOTAsImV4cCI6MjA4MDEzNDI5MH0.1aAeak5leG6uxyMEO7_W_dG94wMf_grxXxUWqfF6iF8"
    "Content-Type"  = "application/json"
  } `
  -Method GE
```
