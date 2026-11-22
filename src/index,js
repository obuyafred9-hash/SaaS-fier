export default {
  async fetch(request, env) {
    const url = new URL(request.url);

    // 1. WEBHOOK ENDPOINT: Receives SMS from Android app or Test Simulator
    if (request.method === "POST" && url.pathname === "/api/webhook") {
      try {
        const { sender, message, secret } = await request.json();

        // Simple security key so random people can't fake payments
        if (secret !== "MY_SHOP_SECRET_123") {
          return new Response(JSON.stringify({ error: "Unauthorized" }), { status: 401 });
        }

        const parsed = parsePaymentSMS(sender || "", message || "");
        if (!parsed) {
          return new Response(JSON.stringify({ error: "Unrecognized SMS format" }), { status: 400 });
        }

        await env.DB.prepare(
          `INSERT INTO payments (provider, tx_code, amount, customer_name, customer_phone, raw_sms)
           VALUES (?, ?, ?, ?, ?, ?)`
        )
          .bind(parsed.provider, parsed.code, parsed.amount, parsed.customer, parsed.phone, message)
          .run();

        return new Response(JSON.stringify({ success: true, payment: parsed }), {
          headers: { "Content-Type": "application/json" }
        });
      } catch (err) {
        return new Response(JSON.stringify({ error: err.message }), { status: 500 });
      }
    }

    // 2. API ENDPOINT: Fetch recent payments for the cashier dashboard
    if (request.method === "GET" && url.pathname === "/api/payments") {
      const { results } = await env.DB.prepare(
        "SELECT * FROM payments ORDER BY id DESC LIMIT 50"
      ).all();
      return new Response(JSON.stringify(results), {
        headers: { "Content-Type": "application/json" }
      });
    }

    // 3. FRONTEND DASHBOARD: Live Cashier Screen + Simulator
    return new Response(renderDashboardHTML(), {
      headers: { "Content-Type": "text/html;charset=UTF-8" }
    });
  }
};

function parsePaymentSMS(sender, text) {
  const cleanSender = sender.trim().toUpperCase();

  // M-PESA Till / Paybill
  if (cleanSender === "MPESA") {
    const m = text.match(/^([A-Z0-9]+)\s+Confirmed\.\s+Ksh([\d,]+\.\d{2})\s+received\s+from\s+(.+?)\s+(\d{10,12})/i);
    if (m) {
      return {
        provider: "M-PESA",
        code: m[1].toUpperCase(),
        amount: parseFloat(m[2].replace(/,/g, "")),
        customer: m[3].trim(),
        phone: m[4]
      };
    }
  }

  // Co-opTill / Co-op Bank
  if (cleanSender.includes("CO-OP") || cleanSender.includes("COOP")) {
    const m = text.match(/([A-Z0-9]+).*?KES\s*([\d,]+\.\d{2}).*?from\s+(.+?)(?:\s+on|\.)/i);
    if (m) {
      return {
        provider: "CO-OP TILL",
        code: m[1].toUpperCase(),
        amount: parseFloat(m[2].replace(/,/g, "")),
        customer: m[3].trim(),
        phone: "N/A"
      };
    }
  }

  return null;
}

function renderDashboardHTML() {
  return `<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>Universal Till Verifier</title>
  <style>
    body { font-family: system-ui, sans-serif; background: #0f172a; color: #f8fafc; margin: 0; padding: 20px; }
    .header { display: flex; justify-content: space-between; align-items: center; margin-bottom: 20px; }
    .badge { background: #16a34a; padding: 6px 12px; border-radius: 999px; font-size: 14px; font-weight: bold; }
    .sim-bar { display: flex; gap: 10px; margin-bottom: 20px; flex-wrap: wrap; }
    button { background: #2563eb; color: white; border: none; padding: 10px 16px; border-radius: 8px; cursor: pointer; font-weight: 600; }
    button:hover { background: #1d4ed8; }
    .card { background: #1e293b; border-left: 6px solid #22c55e; padding: 16px; border-radius: 10px; margin-bottom: 12px; display: flex; justify-content: space-between; align-items: center; }
    .amount { font-size: 22px; font-weight: 800; color: #4ade80; }
    .meta { color: #94a3b8; font-size: 14px; margin-top: 4px; }
  </style>
</head>
<body>
  <div class="header">
    <h2>Live Till Verifier</h2>
    <span class="badge">● Live Connected</span>
  </div>

  <div class="sim-bar">
    <button onclick="simulate('MPESA')">+ Simulate M-Pesa (KES 1,250)</button>
    <button onclick="simulate('CO-OP')">+ Simulate Co-opTill (KES 800)</button>
  </div>

  <div id="feed">Loading payments...</div>

  <script>
    let lastId = 0;

    function playChime() {
      const ctx = new (window.AudioContext || window.webkitAudioContext)();
      const osc = ctx.createOscillator();
      osc.type = "sine";
      osc.frequency.setValueAtTime(880, ctx.currentTime);
      osc.connect(ctx.destination);
      osc.start();
      osc.stop(ctx.currentTime + 0.18);
    }

    async function loadPayments() {
      const res = await fetch('/api/payments');
      const data = await res.json();
      if (data.length > 0 && data[0].id > lastId) {
        if (lastId !== 0) playChime();
        lastId = data[0].id;
      }
      document.getElementById('feed').innerHTML = data.map(p => \`
        <div class="card">
          <div>
            <strong>\${p.customer_name}</strong> (\${p.provider})
            <div class="meta">Code: \${p.tx_code} • \${p.created_at}</div>
          </div>
          <div class="amount">KES \${p.amount.toLocaleString()}</div>
        </div>
      \`).join('') || '<p>No payments yet. Click a simulate button above!</p>';
    }

    async function simulate(type) {
      const code = 'S' + Math.random().toString(36).substring(2, 9).toUpperCase();
      const payload = type === 'MPESA'
        ? { sender: 'MPESA', secret: 'MY_SHOP_SECRET_123', message: \`\${code} Confirmed. Ksh1,250.00 received from JOHN KAMAU 0712345678 on 21/11/26 at 8:25 PM.\` }
        : { sender: 'Co-opBank', secret: 'MY_SHOP_SECRET_123', message: \`\${code} Confirmed. KES 800.00 received from MARY WANJIKU on 21/11/26.\` };

      await fetch('/api/webhook', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(payload)
      });
      loadPayments();
    }

    loadPayments();
    setInterval(loadPayments, 3000);
  </script>
</body>
</html>`;
}
