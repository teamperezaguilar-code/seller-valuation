# Valuation API Backend — RentCast Integration

This is the missing piece between the valuation landing page and RentCast's
AVM (automated valuation model) API. It holds your API key privately and
returns a clean value range to the page.

## Why this exists

The landing page's JavaScript runs in the visitor's browser, so anything
written into it (including an API key) is publicly visible via "view
source." This backend keeps the key on a server instead, where only your
code can see it.

## What this is built for

Deploy as a serverless function (Vercel, Netlify, or Cloudflare Workers are
the simplest options, no server to maintain, generous free tiers). The code
below is written for **Vercel/Node.js**, since it requires the least setup.
If your developer prefers a different platform, the logic transfers
directly, only the file structure and export syntax change.

---

## File: /api/valuation.js

```javascript
export default async function handler(req, res) {
  // CORS headers — required so the browser allows the landing page
  // (a different domain) to call this function at all.
  res.setHeader('Access-Control-Allow-Origin', '*');
  res.setHeader('Access-Control-Allow-Methods', 'POST, OPTIONS');
  res.setHeader('Access-Control-Allow-Headers', 'Content-Type');

  // Browsers send an OPTIONS "preflight" request before the real POST,
  // to check permissions. Answer it immediately with no body.
  if (req.method === 'OPTIONS') {
    return res.status(200).end();
  }

  if (req.method !== 'POST') {
    return res.status(405).json({ error: 'Method not allowed' });
  }

  const { address } = req.body;

  if (!address || typeof address !== 'string' || address.trim().length < 5) {
    return res.status(400).json({ error: 'A valid address is required' });
  }

  try {
    const url = new URL('https://api.rentcast.io/v1/avm/value');
    url.searchParams.set('address', address.trim());

    const rentcastResponse = await fetch(url.toString(), {
      method: 'GET',
      headers: {
        'Accept': 'application/json',
        'X-Api-Key': process.env.RENTCAST_API_KEY
      }
    });

    if (!rentcastResponse.ok) {
      const errorText = await rentcastResponse.text();
      throw new Error(`RentCast ${rentcastResponse.status}: ${errorText}`);
    }

    const data = await rentcastResponse.json();

    if (data.price == null) {
      return res.status(404).json({ error: 'No estimate available for this address' });
    }

    // Return the address RentCast actually matched to, so the frontend can
    // show the visitor exactly which property their estimate is for. Their
    // parser is tolerant of malformed input, so what they matched can
    // differ from what was typed.
    const matchedAddress = data.subjectProperty && data.subjectProperty.formattedAddress;

    return res.status(200).json({
      low: data.priceRangeLow,
      high: data.priceRangeHigh,
      value: data.price,
      matchedAddress: matchedAddress || null,
      disclaimer: 'Automated valuation model estimate provided by RentCast. Not an appraisal. Actual market value can only be determined through an in-person evaluation.'
    });

  } catch (err) {
    console.error('Valuation lookup failed:', err);
    return res.status(502).json({ error: 'Unable to retrieve estimate right now' });
  }
}
```

**A limitation worth knowing:** RentCast's value estimate range (`priceRangeLow`/`priceRangeHigh`) represents an 85% confidence interval per their documentation, not a hard guarantee. Worth keeping the existing "preliminary range" framing on the landing page rather than presenting it as precise.

---

## File: package.json

Vercel needs this small file at the root of the project alongside the `api`
folder, even though this function has no external dependencies.

```json
{
  "name": "team-perez-aguilar-valuation-api",
  "version": "1.0.0",
  "private": true,
  "type": "module"
}
```

---

## Frontend connection — status

✅ Already done. `valuation-landing-bilingual.html` calls this endpoint
directly:

```javascript
const VALUATION_ENDPOINT = 'https://seller-valuation-api.vercel.app/api/valuation';
```

The `hashSeed()` mock has been removed from the page. No further frontend
changes are needed for this integration.

---

## Checklist before this goes live

- [ ] RentCast API key retrieved from your account dashboard
- [ ] Function deployed to Vercel (or Netlify/Cloudflare)
- [ ] `RENTCAST_API_KEY` set as an environment variable on the hosting platform, never hardcoded in the file
- [ ] RentCast's AVM disclaimer language (already included in the code above) visible near the estimate on the landing page
- [ ] Tested against 10-15 known addresses for believable ranges
- [ ] Confirmed a "no match found" address shows a friendly message, not a broken page
- [ ] Check your RentCast plan's monthly call limit before this goes live with ad traffic
- [ ] CORS configured if the landing page and backend are on different domains (Vercel handles this automatically for same-project deployments)
