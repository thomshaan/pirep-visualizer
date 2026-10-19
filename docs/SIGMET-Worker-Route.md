# Worldwide SIGMET endpoint for PIREP Visualizer

The visualizer's SIGMET toggle calls:

`GET https://pirep-metar-api.thomassiahaan31.workers.dev/sigmet`

The endpoint should return a GeoJSON `FeatureCollection` from the Aviation Weather Center's worldwide SIGMET feed. The source lists SIGMET coverage as worldwide and offers GeoJSON: https://aviationweather.gov/data/api/

## Add this route to the existing Cloudflare Worker

Keep the existing METAR route. Add this branch **inside your existing fetch handler, before its current METAR/default route handling**. Do not replace the whole Worker, because that could break the working `/metar?icao=...` endpoint.

```js
const url = new URL(request.url);

if (request.method === "OPTIONS") {
  return new Response(null, {
    headers: {
      "Access-Control-Allow-Origin": "*",
      "Access-Control-Allow-Methods": "GET, OPTIONS",
      "Access-Control-Allow-Headers": "Content-Type, Accept",
      "Access-Control-Max-Age": "86400"
    }
  });
}

if (url.pathname === "/sigmet" && request.method === "GET") {
  const upstream = await fetch(
    "https://aviationweather.gov/api/data/sigmet?format=geojson",
    {
      headers: {
        "Accept": "application/geo+json, application/json",
        "User-Agent": "PIREP-Visualizer/1.0 (worldwide SIGMET overlay)"
      },
      cf: { cacheTtl: 60, cacheEverything: true }
    }
  );

  // The upstream can return 204 when no current SIGMET data is available.
  if (upstream.status === 204) {
    return new Response(JSON.stringify({
      type: "FeatureCollection",
      features: []
    }), {
      status: 200,
      headers: {
        "Content-Type": "application/geo+json; charset=utf-8",
        "Access-Control-Allow-Origin": "*",
        "Cache-Control": "public, max-age=60"
      }
    });
  }

  const body = await upstream.text();
  return new Response(body, {
    status: upstream.status,
    headers: {
      "Content-Type": upstream.headers.get("Content-Type") || "application/geo+json; charset=utf-8",
      "Access-Control-Allow-Origin": "*",
      "Access-Control-Allow-Methods": "GET, OPTIONS",
      "Access-Control-Allow-Headers": "Content-Type, Accept",
      "Cache-Control": "public, max-age=60"
    }
  });
}
```

If the existing Worker already handles `OPTIONS`, keep its existing CORS handling rather than adding a duplicate. The route must run before any fallback that returns the METAR response.

## Test the endpoint

Open:

`https://pirep-metar-api.thomassiahaan31.workers.dev/sigmet`

A healthy response should be a GeoJSON `FeatureCollection` with a `features` array. Each feature should have a `geometry` with `Polygon`, `MultiPolygon`, `LineString`, or `MultiLineString` coordinates. If the response is an error object, inspect the Worker logs and the upstream status before troubleshooting the globe.

## Notes

- The visualizer requests this endpoint only when the SIGMET toggle is enabled and refreshes no more often than every five minutes.
- The overlay draws geographic boundary outlines and provides a hover tooltip where source properties are available.
- This is a situational-awareness display, not an official flight-safety or dispatch tool. Check official aviation weather briefings for operational decisions.
