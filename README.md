# bigtop
(Experimental) Xaringan with configurable styles

## Concept

Xaringan and other R Markdown-based presentation formats are great for quickly and reproducibly building presentations, but if you want to customise colours of fonts, you need to dip into the sometimes-scary world of [Cascading Style Sheets (CSS)](https://en.wikipedia.org/wiki/Cascading_Style_Sheets).

The idea with {bigtop} is to create a Xaringan theme that leverages [CSS custom properties](https://developer.mozilla.org/en-US/docs/Web/CSS/Using_CSS_custom_properties) to make limited customisations available right from the frontmatter of the RMarkdown document, like:

```yaml
---
title: "My sweet slide deck"
author: "James Goldie"
styles:
  fonts:
    header:
      - "Montserrat"
      - "Helvetica Neue"
    body:
      - "Helvetica Neue"
      - "Arial"
      - "sans"
  colors:
    - red
    - green
    - yellow
---
```

## Inplementation

Here's how CSS custom properties can work:

`style.css`

```css
h1 {
  color: red;
  background-color: var(--main-bg-color);
}
```

`test.html`

```html
<html>
  <head>
    <title>Test test test</title>
    <link rel="stylesheet" type="text/css" href="style.css">
    <!-- insert css variables here! -->
    <style>
      :root {
        --main-bg-color: pink;
      }
    </style>
  </head>
  <body>
    <h1>Test test test</h1>
    <p>This'll help me work it out!</p>
  </body>
</html>

```

Here, the CSS file describes the style of the `<h1>` element in the HTML—but its background color is set to a CSS custom property. That property is then defined in a `<style>` tag inside `<head>`.

The benefit of this approach is that the CSS file doesn't need to be touched. It's possible that a properly specified HTML template could allow CSS info to be injected without even using the pre- or post-render hooks in RMarkdown (although I suspect this approach will be a little limited, in that no processing of the frontmatter values will be possible).

(~~Actually, I'm not sure that the HTML template needs to be modified either: CSS custom properties even work if the property definition comes before the included CSS file. This means I could use the `includes` argument to `html_document`~~... although I'm not sure that's designed to touch the frontmatter. Nope, it's for specifying separate files like stylesheets.)

A more complex process could use the hooks to process the frontmatter before it's inserted into the template. Because `xaringan::moon_reader` hardcodes the HTML template (the one in the package), I'm going to have to reimplement it. That sucks a little.

## Notes

I'm trying to work out how [`rmarkdown::render()`](https://github.com/yihui/xaringan/blob/master/R/render.R) and [`xaringan::moon_reader()`](https://github.com/yihui/xaringan/blob/master/R/render.R) work—I want to see if there's a place I can break in and send YAML to a CSS file (in the way that Rmarkdown content might be sent through an HTML template).
 
### `rmarkdown::render()`
 
I think the action starts around [line 420](https://github.com/rstudio/rmarkdown/blob/7dc3cbc6852b496ca031f4ec9db6f0c8e387941f/R/render.R#L420) of `rmarkdown::render()`, where the YAML from the input file is parsed. Then:

* [line 497](https://github.com/rstudio/rmarkdown/blob/7dc3cbc6852b496ca031f4ec9db6f0c8e387941f/R/render.R#L497): what's an "intermediates generator"?
* [line 530](https://github.com/rstudio/rmarkdown/blob/7dc3cbc6852b496ca031f4ec9db6f0c8e387941f/R/render.R#L530): "call any pre-knit handler" (the only arg this takes is this original input contents... so basically the whole file rmd).
* [line 535](https://github.com/rstudio/rmarkdown/blob/7dc3cbc6852b496ca031f4ec9db6f0c8e387941f/R/render.R#L535): post-knot handler takes `yaml_front_matter` (ie. parsed YAML), `knit_input`, `runtime` and `encoding`.
* Lines 555–763: if `requires_knit` (otherwise, just `call_post_knit_handler()`):
  * [line 673](https://github.com/rstudio/rmarkdown/blob/7dc3cbc6852b496ca031f4ec9db6f0c8e387941f/R/render.R#L673): "make the params available within the knit environment" (this is specifically the user-defined `params`, not the entire YAML front-matter)
  * [line 712](https://github.com/rstudio/rmarkdown/blob/7dc3cbc6852b496ca031f4ec9db6f0c8e387941f/R/render.R#L712): "make the yaml_front_matter available as 'metadata' within the knit environment"
  * [line 727](https://github.com/rstudio/rmarkdown/blob/7dc3cbc6852b496ca031f4ec9db6f0c8e387941f/R/render.R#L727): "call onKnit hooks"
  * [line 740](https://github.com/rstudio/rmarkdown/blob/7dc3cbc6852b496ca031f4ec9db6f0c8e387941f/R/render.R#L740): "perform the knit"
* A bunch of Pandoc stuff that I probably don't care about (Xaringan uses inline Markdown, not Pandoc)
* [line 950](https://github.com/rstudio/rmarkdown/blob/7dc3cbc6852b496ca031f4ec9db6f0c8e387941f/R/render.R#L950): "if there is a post-processor then call it" (it takes `yaml_front_matter`, `utd8_input`, `output_format`, `clean` and `!quiet`)
* That's it!

### `rmarkdown::output_format()`

Okay, functions like `xaringan::moon_reader()` actually return the result of a call to `rmarkdown::output_format()` (which I think is why you see references in `render()` to things like `output_format$pre_knit` etc.). This is where format-specific things like `pre_knit`, `post_processor`, etc. are defined.

https://github.com/yihui/xaringan/blob/85c0e7f268ba94306bb33f2cfb70ad846d03b9ad/R/render.R#L165

`...` (which comes all the way from `rmarkdown::render()` at the top) is passed to the `pre_knit` function and also to `rmarkdown::html_document2`, which is the `base_format`. According to `html_document`, `extra_dependencies` and `...` are "Additional function arguments to pass to the base R Markdown HTML output formatter `html_document_base`". But that function ignores `...`.

I don't think I can pass anything all the way in from `rmarkdown::render()`. But maybe I can fork `moon_reader()` or make a version of it that includes some extra code in `pre_processor`.
