+++
title = "Astro Cal: A Rust WASM App for Astronomical Calendar Events"
date = 2022-12-20
image = "assets/flower_in_rocks-1200px.jpg"
image_alt = "Picture of flowers blooming amount rocks"
summary = """
Building a Rust WASM app with Yew to generate calendar events for astrological events such as Moonrise/Moonset.
"""
source_code = "github.com/whynotcats/astro-events"
view_site = "astro-events.whynotcats.com"
project_name = "Astro Cal"
[taxonomies]
tags = ["rust", "yew", "bulma", "elasticsearch"]
+++

## Externalizing memory

One my on-going projects is to externalize as much of my memory as I can. I've been doing this with services like [Pinboard](https://pinboard.in) for recipes and blog posts, and [1password](https://1password.com) for passwords, software licenses, etc. One more thing I would like to be able to quickly know _and_ get notified on is when moonrise is. I've been fortunate enough to by chance have seen very cool moonrises such as a full-moon rising behind Mt Baker and a harvest moon over the Vancouver skyline. Both times I didn't have my camera on me, so I'd love to be able to better plan for it.

There are a number of apps that have information about ideal photo conditions and astrological events such as [Photo Pills](https://www.photopills.com/) and [Night Sky](https://apps.apple.com/us/app/night-sky/id475772902). Each of these have an element of what I'm looking for, though none have everything, and all are not incentivized to develop and support the features I want. Having to switch between different apps and relying on app notifications is not something I want either, and for something that happens at predictable intervals/times calendar events should be more than enough. Calendars are an easy thing to check in the morning to see what obligations I have and at work it's a great way for coworkers to book time with me without having to go back a forth a bunch (in work cultures that emphasize using and respecting each others time).

For an MVP I want to focus on calculating moonrises and generating an `.ics` file with events which can be imported into Google Calendar or iCalendar should be good enough. All I need is a Rust frontend framework, a way to calculate moonrise for an arbitrary location, and a way to write to a calendar file.

## Frontend Framework

Being experienced with React it would be nice to use something with similar concepts. [Yew](https://yew.rs/) seems to fit the bill, with a similar Components, state management, and an `html!` for a JSX-like experience. For MVP I don't need some of the more advanced features such as Routing, Suspense, and Server Side Rendering, but it's good to know that they're available. One downside is there doesn't seem to be a great way of currently including HTML syntax highlighting / formatting in `html!` blocks in most editors.

For styling the frontend I wanted something that had some pre-made components I could use to quickly build, but without a requiring javascript. [Bulma](https://bulma.io/) seems to fit the bill, and it has sass source files which I can throw at trunk to build for me when I want to customize things, or just point at the cdn url for the latest compiled css file. A quick [panel](https://bulma.io/documentation/components/panel/) for the location search bar and [card](https://bulma.io/documentation/components/card/) for the results will be more than enough.

Note: There are issues with how I handled the state in the MVP causing generated anchor tags from a Vec of results to stick around instead of clearing when making multiple queries but I'm leaving debugging that for when I refactor into proper pure components instead of one big struct component.

## Geolocation Data

To calculate the times for moonrise I need geolocation coordinates that we'll be viewing from. Applications usually query an API such as Google Maps or [OpenWeatherMap](https://openweathermap.org/) to convert an address or town into a lat-long pair. These often have a free tier giving a small number of queries before charging you and they return more information than I need for such a simple task. I first looked at [GeoNames](https://geonames.org) since they had an API but realized they offered [data dumps](http://download.geonames.org/export/dump/) of geo data that I could just use locally. The data is simple enough: a file of geolocation data and a few meta files to enrich the data. For quick access to a relatively static dataset I considered trying redis server's [full text search](https://redis.com/redis-best-practices/indexing-patterns/full-text-search/) indexing but decided to go with a more familiar elasticsearch, since I can just dump documents and get full indexing. I'll also likely need a quick search index for future projects, so it makes sense for me to set it up. A simple fuzzy query filtering out non-populated locations should be give me good enough results:

```javascript
{
  "query": {
    "bool": {
      "must": {
        "multi_match": {
          "fields": ["name", "country_code"],
          "query": "vancouver",
          "fuzziness": "AUTO"
        }
      },
      "filter": {
        "range": { "population": { "gt": 0 } }
      }
    }
  }
}
```

I setup a small CLI app to load the data into elasticsearch from the files ([source](github.com/whynotcats/admin)). This data isn't likely to change very often, so developing a process to automatically and/or incrementally updates the data shouldn't be necessary. Most people would only want the precision of their city, and cities aren't created/moved that often. [Axum](https://github.com/tokio-rs/axum) + [Tower](https://github.com/tower-rs/tower) seems to be more than enough to serve as a basic REST API to serve both searching for locations and generating the calendar data. Not much to say there, works similarly to Node.js REST frameworks like [Express](https://expressjs.com/).

## Astro library

For doing that actual calculations for moonrise [astro-rust](https://github.com/saurvs/astro-rust) seems to have a robust set of astronomical math functions allowing for things like filtering by the location of moonrise in the sky. This is unnecessary for MVP and it doesn't have a convenient function to quickly calculate moonrise with a lat, long, and date. One day I will brush up on my space math and reimplement in [astro-rust](https://github.com/saurvs/astro-rust), but for now [geodate](https://github.com/vinc/geodate) has the basic functionality I need.

Only issue that came up was some weird possible edge case when a daylight savings time change happened at the same time moonrise crossed `00:00 UTC`, at least that's the most proximate cause I can think of. When I was generating calendar events for a set of days that crossed the time the DST change happened it generated two events on the same day just minutes apart. I tried poking around at this issue for a while but for now I just threw in a check against the previous moonrise to make sure they aren't too close:

```rust
    if next_moonrise.is_some() && next_moonrise.unwrap() - previous_moonrise <= 500 {
        next_moonrise = get_moonrise(julian_to_unix(jd + 1.), lon, lat);
    }
```

## Calendar Library

For my first pass at this the library [iCalendar](https://docs.rs/icalendar/latest/icalendar/) works well enough for my purposes. I can create a calendar with an arbitrary amount of events and write to a file/buffer which I can then import into google calendar manually later. It would be nice to have built-in google calendar integration and the ability to update an existing calendar, but that can be put into todos for later. For event type and length I'm going with a default of a 15 minute event that starts 15 minutes before calculated moonrise. I want to be able to get a calendar alert notification with enough time to stop what I'm doing and go outside to view the moonrise, but I'll need to tweak this time as the mountains around me push the visible moonrise back.

## Shipped

All in all putting developing the app was straightforward and easy, the parts that took the longest were the CSS and the rabbit-hole of setting up hosting. As simple as it is the initial app has bugs when daring to step outside the happy path - I didn't even implement debouncing API calls on `keyup` for locations, just used a simple form with `preventDefault()` - but that's the joy of developing for yourself. Try it at [astro-cal.whynotcats.com](https://astro-cal.whynotcats.com).
