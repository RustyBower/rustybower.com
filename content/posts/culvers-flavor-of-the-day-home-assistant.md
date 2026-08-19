---
title: "A Culver's Flavor of the Day Dashboard, and the Sanitizer That Ate My CSS"
date: 2026-10-06
draft: false
categories: ["home-automation", "dashboards", "debugging"]
tags: ["home-assistant", "lovelace", "card-mod", "mushroom"]
---

Culver's rotates a different frozen custard flavor every day at every location, and
the flavors are not synchronized between stores. Four restaurants within a
reasonable drive means four different answers to "is it worth going tonight," and
the only way to find out has been to open the app four times.

So it went on the wall tablets. The data part took one commit. The dashboard took
six, and five of them were wrong for a reason I could not see.

## The API that isn't an API

Culver's publishes no developer API, but their store locator is a plain unauthenticated
JSON endpoint that the website itself calls, and it returns everything needed in a
single request:

```
GET https://www.culvers.com/api/locator/getLocations?lat=<lat>&long=<long>&radius=40000&limit=4
```

Two quirks cost me more time than the rest of the integration combined.

**The parameters must be named exactly `lat` and `long`.** Not `latitude`, not
`location`, not `q`. Anything else returns a success-shaped failure:

```json
{"isSuccessful":false,"message":"No location data","data":{"meta":{"code":200}}}
```

Note the `"code":200` nested inside. HTTP 200, `isSuccessful: false`, no data. If you
are only checking status codes you will conclude there are no Culver's in Wisconsin.

**`radius` is in meters.** Passing `25` on the assumption it meant miles is also a
200, also well-formed, and also empty:

```
totalResults: 0    geofences: 0
```

`40000` is about 25 miles, which from a typical Madison-area address reaches 21 stores.

What comes back is one `geofence` object per restaurant, sorted nearest-first, each
carrying the store's flavor of the day inline:

```
'Madison, WI - Todd Dr'
   flavor: OREO® Cookie Overload
   slug:   img-Oreo-Overload.png
   addr:   2102 West Beltline Hwy., Madison, WI
   desc:   Our delicious Chocolate Fresh Frozen Custard loaded with OREO® Cookies...

'Madison, WI - Northport Dr'
   flavor: Mint Cookie
   slug:   img-Mint-Oreo-updated.png

'Madison, WI - Cottage Grove Rd'
   flavor: Caramel Chocolate Pecan
   slug:   img-Caramel-Chocolate-Pecan3.png

'Madison, WI - Mineral Point Rd'
   flavor: Caramel Turtle
   slug:   img-Caramel-Turtle.Waffle-Cone.png
```

The `flavorOfDaySlug` prefixed with `https://cdn.culvers.com/menu-item-detail/` gives
you the product photo — a ~300 KB transparent 800x800 PNG. Flavor name, description,
photo, address, and coordinates, all from one GET. For a service with no API this is
an unusually good one.

### Leaving 39% of the payload on the floor

Each geofence also ships the store's delivery-area polygon. For a four-store response
that is 5,578 bytes of coordinates out of 14,487 total — **39% of the payload is
polygons I will never draw**. Anything pulled into `json_attributes` lands in the state
machine and then in the recorder database, every 30 minutes, forever, so the sensors
name their attributes explicitly and `geometry` is simply not on the list:

```yaml
rest:
  - resource_template: >-
      https://www.culvers.com/api/locator/getLocations?lat={{ state_attr('zone.home', 'latitude') }}&long={{ state_attr('zone.home', 'longitude') }}&radius=40000&limit=4
    scan_interval: 1800
    headers:
      user-agent: Home Assistant
    sensor:
      - name: Culvers Store 1
        unique_id: culvers_store_1
        icon: mdi:ice-cream
        value_template: >-
          {% set gf = value_json.get('data', {}).get('geofences', []) %}
          {{ gf[0].metadata.flavorOfDayName if gf | count > 0 else 'Unknown' }}
        json_attributes_path: "$.data.geofences[0]"
        json_attributes: &store_attrs
          - description
          - geometryCenter
          - metadata
```

`geometryCenter` is kept, and it is the one that earns its place: it lets the template
below compute real driving-adjacent distance from home rather than trusting the API's
sort order.

### One sensor per store, then folded back into one

There are four nearly identical `rest` sensors, one per store, differing only in the
array index. That looks like something a loop should handle, and the reason it can't is
`json_attributes_path`: it addresses exactly one object per sensor. A single sensor
holding the whole payload could expose the flavors but not each store's attributes
separately.

So the four are helpers, and a template sensor folds them into the one entity the
dashboard actually consumes:

```yaml
template:
  - sensor:
      - name: Culvers Flavors
        unique_id: culvers_flavors
        state: "{{ states('sensor.culvers_store_1') }}"
        attributes:
          locations: >-
            {% set ns = namespace(items=[]) %}
            {% for e in ['sensor.culvers_store_1', 'sensor.culvers_store_2',
                         'sensor.culvers_store_3', 'sensor.culvers_store_4'] %}
              {% set m = state_attr(e, 'metadata') %}
              {% set c = state_attr(e, 'geometryCenter') %}
              {% if m %}
                {% set ns.items = ns.items + [{
                     'name': state_attr(e, 'description'),
                     'flavor': states(e),
                     'description': m.flavorOfTheDayDescription | default('', true),
                     'image': ('https://cdn.culvers.com/menu-item-detail/' ~ m.flavorOfDaySlug)
                              if m.flavorOfDaySlug else '',
                     'address': m.street ~ ', ' ~ m.city ~ ', ' ~ m.state,
                     'miles': ((distance(c.coordinates[1], c.coordinates[0]) or 0) | round(1))
                              if c else 0,
                     'closed': m.isTemporarilyClosed | default(false, true) }] %}
              {% endif %}
            {% endfor %}
            {{ ns.items }}
```

`coordinates` is GeoJSON, so it is `[long, lat]` and `distance()` wants `(lat, long)` —
hence the reversed indices. The result is `sensor.culvers_flavors` with a clean
`locations[]` list the dashboard loops over.

One deliberate choice here: this uses the `rest` platform rather than `rest_command`
plus a trigger template. Both can express the same thing, but `rest_command` is not a
loaded domain in this instance, so adding it would have made the package require a full
Home Assistant restart on every edit. As written it reloads with the `rest.reload` and
`template.reload` actions, which matters a lot when you are about to iterate on the
frontend six times.

## The dashboard, take one through five

The obvious card for "loop over a list of things" is a markdown card, since it templates
its own content. So: a table of stores, each with a photo, flavor, distance, and
description, styled with inline CSS.

It rendered with **enormous** photos. Full column width, one store per screen on the
tablets.

I tuned the grid. `repeat(auto-fit, minmax(230px, 1fr))` let tiles grow to fill the row,
so I pinned it to `repeat(4, 1fr)` and added `min-width: 0` — grid items default to
`min-width: auto` and refuse to shrink below their content. Still enormous. I scaled the
photos down explicitly. Still enormous.

Every fix I made was correct and none of them did anything, which is the specific
frustration that eventually makes you go read the card's source.

### The markdown card sanitizes with js-xss

Home Assistant's markdown card runs its rendered output through
[js-xss](https://github.com/leizongmin/js-xss) before it reaches the DOM, and the
whitelist is per-element and strict. `div` and `span` allow **no attributes at all**.
`img` allows `[src, alt, title, width, height, loading]`. `style` and `class` are on no
element's list.

Every inline style I had written in three consecutive revisions was being silently
stripped in transit. The photos were not oversized because my CSS was wrong — they were
at their natural 800×800, constrained only by the card's own `img { max-width: 100% }`,
because my CSS was never there.

Nothing warns you. The markdown goes in, the styles come out, and the card renders
something plausible enough that you blame yourself.

### Rebuilding with what survives

The fix was to stop writing CSS and start writing 1998 HTML — the attributes on the
whitelist are the ones that work:

- a `<table>` for the columns, with `width=` on the cells
- `width=104` on each `<img>`
- 2% spacer cells for gutters, because the card sets `table { border-collapse: collapse }`,
  which ignores `border-spacing`

That rendered correctly. It also looked like 1998, so the next step was installing
[card-mod](https://github.com/thomasloven/lovelace-card-mod), which injects styles
directly into `ha-markdown`'s shadow root — downstream of the sanitizer entirely. That
restored the tile look, and the plain table left in the content became a genuine
fallback: if card-mod ever fails to load, the card degrades to a readable table of 104px
photos rather than 800px ones.

One more layout casualty followed. Each store had a rounded background panel, which read
as a box nested inside the card's own rounded box, and adjacent panels butted together
because `border-collapse: collapse` was still ignoring the gutters. Dropping the panels
fixed both problems at once.

## Take six: stop fighting the card

By this point the card was styled correctly and still wrong in two ways that no amount
of CSS could fix.

First, a markdown card is *one* `ha-card`. Four stores drawn as columns inside it read
as tiles nested in a tile no matter how they are styled.

Second, and fatal: **sanitized markdown content cannot carry a `tap_action`.** The only
click a markdown card can offer is a plain `<a>`, and the wall tablets run Fully Kiosk,
which will not follow a link off-dashboard. Tapping a flavor to see the full description
was the entire point of tapping a flavor.

So the markdown card went away, replaced by four `mushroom-template-card`s in a
`layout-card` grid. These are real cards, so the grid separates them with actual card
chrome, and the tap fires `browser_mod.popup` and never navigates:

```yaml
- type: custom:mushroom-template-card
  layout: vertical
  multiline_secondary: true
  picture: >-
    {% set l = (state_attr('sensor.culvers_flavors', 'locations') or [])[0:1] | first | default({}, true) %}
    {{ l.get('image', '') }}
  primary: >-
    {% set l = (state_attr('sensor.culvers_flavors', 'locations') or [])[0:1] | first | default({}, true) %}
    {{ l.get('flavor', '—') }}
  secondary: |-
    {%- set l = (state_attr('sensor.culvers_flavors', 'locations') or [])[0:1] | first | default({}, true) -%}
    {{ l.get('name', '').split(',')[0] }} · {{ l.get('miles', 0) }} mi
    {{ l.get('description', '') }}
  tap_action:
    action: fire-dom-event
    browser_mod:
      service: browser_mod.popup
      data:
        title: Flavor of the Day
        content:
          type: markdown
          content: |-
            {%- set l = (state_attr('sensor.culvers_flavors', 'locations') or [])[0:1] | first | default({}, true) -%}
            <img src="{{ l.get('image', '') }}" width="280">

            ### {{ l.get('flavor', '—') }}

            **{{ l.get('name', '') }}** · {{ l.get('miles', 0) }} mi

            {{ l.get('description', '') }}

            {{ l.get('address', '') }}
  card_mod:
    style: |
      ha-card {
        --icon-size: 116px;
        --icon-border-radius: 8px;
      }
```

A few things worth extracting from that:

**The popup body is a markdown card, and it has to be.** `browser_mod` does not template
its service data, so `{{ }}` in the popup's `data` would arrive as a literal string. A
markdown card templates its *own* content, so the flavor has to be resolved inside the
popup card rather than passed into it.

**`--icon-border-radius: 8px` is not cosmetic.** Mushroom's avatar slot is a circle by
default, and a circle clips the corners off a waffle cone cutout.

**`[0:1] | first | default({}, true)` instead of `[0]`.** Indexing an empty list throws
and takes the whole card down; the slice-then-default idiom yields an empty dict and the
card renders blank instead of broken.

The grid is one column below 600px and four above, so phones stack and the wall tablets
get a single row.

## Caveats

**The store count is now hardcoded at four.** The markdown version looped over the sensor,
so changing `limit=` in the REST call was a one-line change and the card followed. The
four-card version does not. Adding a fifth store means a fifth REST sensor, a fifth entity
in the template list, and a fifth copy of a 40-line card. The `default({}, true)` guard
keeps a card blank rather than broken if the locator returns fewer than four, but it
cannot invent a card for a fifth. This was a deliberate trade for real card chrome and a
working `tap_action`, and it is the thing I would revisit first.

**This depends on an endpoint nobody promised me.** It is the store locator Culver's own
site calls, so it is unlikely to vanish quietly, but there is no contract here. If the
response shape changes, the `value_template` guards degrade to `Unknown` rather than
throwing — that is the extent of the safety net.

**The 30-minute poll is more than necessary.** The flavor of the day changes once per day
per store. `scan_interval: 1800` is 48 requests a day against someone else's server for
data that changes once. A daily poll shortly after opening would be the polite version;
I have not made that change yet.

**js-xss stripping is silent by design, and it applies to more than this card.** If you
have ever written inline styles into a Home Assistant markdown card and quietly concluded
you are bad at CSS, you were not — check the whitelist before you check yourself.
