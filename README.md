# websan
Web sanitizer.

## urlsan

A URL/URI should identify a resource.
As an example, assume we want to group browser history by time spent.
We could group by host name, but what if we want to be more granular?
The main problem is that nowadays the URLs used to reach a resource are not necessarily unique, mainly because of added tracking parameters.
So could we group by everything before `?foo=bar&bar=baz`? Not really because we don't know which of the parameters are needed to identify the resource.
We could maintain a list of parameters to strip from URLs to normalize them (`glcid, fbclid, utm_*`, [UTM](https://en.wikipedia.org/wiki/UTM_parameters)), which might be viable for Google Analytics and Facebook, but many websites also add their own parameters representing state or source (e.g. what the user searched for).

The relation (visited URL:content) over time might be
1. 1:1, same URL, same content
2. n:1, multiple URLs, same content
3. 1:n, same URL, content changes

where the levels of content equality could be abstracted as

1. same response to initial GET, e.g. same HTML
2. same DOM after rendering is done, e.g. in puppeteer
3. same content for all URLs in DOM (limit depth, e.g. only for images)

assuming that (not n) implies not (n+1).

The idea is to remove parts as long as the content stays the same:

~~~ocaml
open Batteries open List

let normalize url =
  let res = get url in (* HTTP GET response *)
  let dom = browse url in (* DOM from browser (dynamic parts) *)
  (* since we have different separators between parts, we need to extract them as well? *)
  let parts = split url in (* e.g. ["https://"; "amazon"; "."; "com"; "/"; "123"; "/"; "foo"; "?"; "arg1=val1"; "&"; "gclid=123"] *)
  (* filter for valid URLs to avoid 2^n requests for n parts. *)
  let urls = filter valid (sub_lists parts)
    (* sort by length of url and stop on first match? *)
    |> filter (get %> (=) res)
    |> filter (browse %> (=) dom)
  in combine (map String.length urls) urls |> min |> snd
~~~

We could also try to
- classify static/changing content (Precision would be time-dependent - if content changes during normalize, we know for sure, but if not, we only know it's static for that time.)
- check permutations of parameters since the order is usually not relevant
