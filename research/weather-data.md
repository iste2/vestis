# Weather data for daily recommendations and trip plans

Research for issue #10 (map #2). Researched 2026-11-10 against provider first-party documentation. Items marked **[unverified]** could not be confirmed from a primary page in this session.

## Question

Which weather data source should Vestis use for (a) current and forecast weather at the user's location (**Daily recommendation**) and (b) forecast or climate normals at a trip destination, possibly beyond the forecast horizon (**Trip plan**)? Compare coverage, commercial-use terms, rate limits, cost and attribution. Context: v1 is a free public app with Flutter + Firebase; domain logic lives once in the backend ([ADR-0001](../docs/adr/0001-domain-logic-lives-once-in-the-backend.md)), so weather would be fetched server-side.

## Summary

| | Open-Meteo | Google Maps Platform Weather API | Apple WeatherKit | MET Norway Locationforecast | OpenWeather One Call |
|---|---|---|---|---|---|
| Current + forecast | yes | yes | yes | forecast (current ~ nearest hour) | yes |
| Max forecast horizon | 16 days (model-dependent, 7 default) | 10 days daily, 240 h hourly | [unverified] ~10 days | 9 days | [unverified] |
| Beyond horizon | Historical (ERA5 from 1940), Climate API (CMIP6 1950-2050), Seasonal (to 9 months) | none (history only 24 h) | none found | none | [unverified] |
| Free tier | Non-commercial only; 10k/day, 5k/h, 600/min, 300k/month per IP/app | 10,000 billable events/month, then paid | 500k calls/month with Apple Developer Program | free incl. commercial (CC BY 4.0 / NLOD), <20 req/s total | 1,000 calls/day free (One Call 4.0 page) |
| Paid | Standard 1M/mo, Pro 5M/mo (price not on public page) | $0.15 / 1k (10k-100k), tiers down to $0.038 / 1k | $49.99 per 1M/mo | n/a | $0.0015 / call |
| Attribution | link "Weather data by Open-Meteo.com" next to displayed data | "Source: Includes weather data from Google." or logo, strict styling | Apple Weather trademark + legal link | credit MET Norway | ODbL listed |
| Global | yes | yes except China, Cuba, Iran, North Korea, Syria | yes (regional feature gaps) | yes (best in Nordics) | yes |

## Findings per provider

### Open-Meteo

- **Forecast:** up to 16 days (`forecast_days` max 16, default 7); `past_days` 0-92. Best Match blends DWD ICON, NOAA GFS and ECMWF IFS globally; individual models are shorter (ICON 7.5 d, ECMWF IFS 15 d, GFS 16 d). Source: https://open-meteo.com/en/docs
- **Beyond the horizon, three tools:**
  - Historical Weather API: ERA5 reanalysis from 1940 (0.25 deg, hourly, ~5 days delay), ERA5-Land from 1950. Usable to compute climate normals (e.g. mean/percentile temperature and precipitation for the destination dates over the last 10-30 years). Source: https://open-meteo.com/en/docs/historical-weather-api
  - Climate API: seven CMIP6 HighResMIP models downscaled to 10 km, 1950-2050, endpoint `climate-api.open-meteo.com`. Data is model output "for model validation, not actual measurements"; CC BY 4.0, must credit CMIP6. Source: https://open-meteo.com/en/docs/climate-api
  - Seasonal Forecast API: ECMWF SEAS5 to 9 months, EC46 to 6 weeks; probabilistic, not bias-corrected. Source: https://open-meteo.com/en/features
- **Terms:** "You may only use the free API services for non-commercial purposes." Non-commercial examples include "private or non-profit websites or apps that do not have subscriptions or advertising"; commercial examples include "websites or apps that have subscriptions or display advertisements" and "commercial products or promotional activities". Voluntary donations are allowed. Source: https://open-meteo.com/en/terms
- **Limits (free):** fewer than 10,000 calls/day, 5,000/hour, 600/minute, 300,000/month; applied "by IP addresses and application"; shared-IP platforms (Vercel, Netlify, Render named) may be rate-limited earlier; no uptime guarantee; abusive IPs may be blocked without notice. Same source.
- **Paid:** Standard (1M calls/month), Professional (5M, adds historical, climate, ensemble, satellite radiation data), Enterprise (50M+); unlimited per-minute/hour/day; commercial use allowed; API key on `customer-api.open-meteo.com`; 99.9% uptime target. Monthly price is not shown on the public pricing page (fixed monthly price, see customer portal). Monthly limits are currently not enforced, only alerts at 80/90/100%. Source: https://open-meteo.com/en/pricing
- **Attribution:** CC BY 4.0; "You must include a link next to any location Open-Meteo data are displayed", example `<a href="https://open-meteo.com/">Weather data by Open-Meteo.com</a>`. Upstream sources (DWD, ECMWF, NOAA, etc.) are open-data under CC-BY or similar licences; UK Met Office data is CC-BY-SA. Source: https://open-meteo.com/en/licence
- **Operator:** OpenMeteo GmbH, Swiss law, GDPR-compliant, open-source (AGPLv3) and self-hostable. Source: https://open-meteo.com/en/terms

### Google Maps Platform Weather API

- Endpoints: current conditions, hourly forecast (240 h), daily forecast (10 days), hourly history (24 h). Source: https://developers.google.com/maps/documentation/weather/overview
- Coverage: worldwide except China, Cuba, Iran, North Korea, Syria (Vietnam alerts only); Japan and South Korea from Weathernews, without hourly history. Source: https://developers.google.com/maps/documentation/weather/coverage
- Pricing: single "Weather Usage" SKU, 10,000 free events/month, then $0.15 per 1,000 (10,001-100,000), $0.12, $0.09, $0.06, $0.038 at higher volume tiers; incremental tiers; usage aggregated per billing account. Source: https://developers.google.com/maps/billing-and-pricing/pricing (page carries no date).
- Policies: attribution "Source: Includes weather data from Google." or Google Maps logo, with strict logo/text/placement rules; derived content (e.g. a custom score) must say "Includes data from Google Maps". Caching governed by the Maps Platform Service Specific Terms; the policy page states "Don't pre-fetch, index, store, or redistribute Google Maps Content" and "don't create content based on Google Maps Content". Caching periods were not retrievable. Source: https://developers.google.com/maps/documentation/weather/policies
- No climate normals or long-range product.

### Apple WeatherKit

- 500,000 calls/month included with Apple Developer Program membership; paid plans from $49.99 per 1M/month. Attribution requires the Apple Weather trademark and a legal-sources link; derived "value-added" data must state it was modified by Apple data. REST API exists for non-Apple platforms (JWT auth **[unverified]**; the REST reference page did not render, so datasets and horizon are unconfirmed). Source: https://developer.apple.com/weatherkit/get-started/
- Strong free quota but no climate or long-range product found, and an Apple-ecosystem dependency for an app that also targets Android.

### MET Norway Locationforecast

- Global, 9-day forecast, updated hourly in the Nordics and every 6 h elsewhere. Data under NLOD 2.0 and CC BY 4.0; credit MET Norway. Requires an identifying User-Agent; keep total traffic under 20 req/s; honour `Expires`/`If-Modified-Since`; "Browsers and mobile apps should not contact the API directly" and a caching backend proxy is expected. The terms page did not explicitly confirm commercial use (CC BY 4.0 implies it is allowed). Sources: https://api.met.no/doc/TermsOfService, https://api.met.no/doc/License, https://api.met.no/weatherapi/locationforecast/2.0/documentation
- Quality is highest in the Nordics; no climate normals.

### OpenWeather

- The pricing page lists One Call API 4.0: first 1,000 calls/day free, then $0.0015 per call, ODbL listed as licence; One Call 3.0 specifics (horizon, commercial terms) were not on the retrieved page **[unverified]**. Source: https://openweathermap.org/price. Not pursued further.

## Analysis

1. **Daily recommendation** needs current + 1-3 day forecast per user location. Every compared provider covers this; differences are terms and quota, not capability. Calls should come from the backend with a cache keyed by a coarse location grid (e.g. ~10 km cell) and hour, so call volume scales with distinct locations rather than users. This also helps MET Norway's and Open-Meteo's fair-use expectations.
2. **Trip plan** needs destination weather for dates that are often months away. Only Open-Meteo offers a free path beyond ~10-16 days (reanalysis-derived normals, CMIP6 climate, seasonal outlook). Google, Apple and MET Norway stop at 9-10 days, so they would need a second provider for climate normals anyway. Plan: forecast API within the horizon; otherwise normals computed from ERA5 history for the same calendar window, shown as "typical weather", and labelled as such.
3. **Commercial-use risk with Open-Meteo.** A free, ad-free, subscription-free app fits the "non-commercial" examples literally, but the app is a public store product and the terms also list "commercial products" as commercial. Unclear for a solo-developer free app; written confirmation from Open-Meteo (info@open-meteo.com) is cheap insurance. Any future monetization (explicitly out of scope for v1) flips this to the paid plan. Historical and Climate APIs on a paid plan require Professional, not Standard.
4. **Rate limits are per IP.** Firebase Cloud Functions / Cloud Run egress IPs are shared, so a free-tier outage-by-neighbour is possible; with cell-level caching, 10k calls/day is ample for v1 scale, but there is no SLA.
5. **Google Weather** is the cleanest commercial licence and cheapest at small volume (10k free events/month), but its "no pre-fetch, no storing, no derived content" language conflicts with backend caching and scoring, and it cannot serve trip dates past 10 days. Attribution rules are the strictest.
6. **Quality:** Open-Meteo is a model-aggregation service (national weather services' open data), not a station network; fine for outfit decisions, which tolerate a few degrees of error.

## Recommendation

- **Primary: Open-Meteo**, called only from the backend behind a thin weather port (so the provider can be swapped without touching the Flutter app or the MCP server), with a location-cell cache. Use Forecast API for horizon <= 16 days, Historical (ERA5) API for normals beyond it.
- **Before launch:** get written confirmation from Open-Meteo that the free, ad-free, subscription-free v1 qualifies as non-commercial; put the link "Weather data by Open-Meteo.com" next to weather shown in the app (daily recommendation and trip plan); budget for the Professional plan if the app is monetized, if the confirmation is refused, or if usage outgrows 300k calls/month.
- **Fallback:** Google Weather API (or WeatherKit/MET Norway) for forecast within 10 days if Open-Meteo is unavailable or disallowed; none replaces the beyond-horizon capability.

## Open risks and questions

- Open-Meteo commercial classification of a free public store app (see 3).
- Open-Meteo paid plan prices are not public on the pricing page; obtain a quote.
- Google caching periods in the Service Specific Terms were not retrieved; check before using Google as fallback.
- WeatherKit REST datasets/horizon and OpenWeather One Call 3.0 terms unverified.
- Trip plans for destinations in unusual climates or multi-leg trips need per-leg normals; CMIP6 data is model output, not measurement.
- Privacy: user locations sent to a third party (Open-Meteo, Swiss GmbH); coarse-cell rounding limits exposure. Belongs with the privacy/GDPR map item.
