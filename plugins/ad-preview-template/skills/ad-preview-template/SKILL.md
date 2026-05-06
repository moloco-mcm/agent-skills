---
name: ad-preview-template
description: Generate valid HTML ad preview templates for MCM Sponsored Display and Sponsored Brands. Use when authoring or debugging templates that contain macro placeholders such as `{{headline.text}}` or conditional sections like `{{#banner.image}}`.
---

# Ad Preview Template Specification

You are assisting a developer who builds ad preview HTML templates. Templates use mustache-style `{{macro}}` placeholders that are replaced with ad asset data at render time.

Templates are static HTML only — no JavaScript execution is permitted. The system renders templates in a sandboxed iframe.

Templates are uploaded via an internal admin tool. To submit a template, provide the completed HTML file to your account representative.

## Macro Reference

### Variable Macros

| Macro                             | Type      | Description                                                       |
| --------------------------------- | --------- | ----------------------------------------------------------------- |
| `{{headline.text}}`               | Text      | Ad headline                                                       |
| `{{custom_text.text}}`            | Text      | Custom text field (e.g., sub-headline, footer)                    |
| `{{cta.text}}`                    | Text      | Call-to-action button text                                        |
| `{{banner.image_url}}`            | Image URL | Banner image (use inside `{{#banner.image}}` section)             |
| `{{banner.video_url}}`            | Video URL | Banner video (use inside `{{#banner.video}}` section)             |
| `{{banner.video_thumbnail_url}}`  | Image URL | Video thumbnail (use inside `{{#banner.video}}` section)          |
| `{{brand_logo.image_url}}`        | Image URL | Brand logo image                                                  |
| `{{catalog_items[N].title}}`      | Text      | Catalog item title (**Sponsored Brands only**, N is 0-based)      |
| `{{catalog_items[N].image_url}}`  | Image URL | Catalog item image (**Sponsored Brands only**, N is 0-based)      |
| `{{catalog_items[N].price}}`      | Text      | Catalog item price (**Sponsored Brands only**, N is 0-based)      |
| `{{catalog_items[N].sale_price}}` | Text      | Catalog item sale price (**Sponsored Brands only**, N is 0-based) |

### Conditional Sections

Conditional sections control the visibility of HTML blocks. Use `{{#section}}` to open and `{{/section}}` to close.

| Section                                   | Renders when                                            |
| ----------------------------------------- | ------------------------------------------------------- |
| `{{#banner.image}}...{{/banner.image}}`   | The banner asset is an image                            |
| `{{#banner.video}}...{{/banner.video}}`   | The banner asset is a video                             |
| `{{#brand_logo}}...{{/brand_logo}}`       | Brand logo data is available                            |
| `{{#catalog_items}}...{{/catalog_items}}` | Catalog items are available (**Sponsored Brands only**) |

Include **both** `{{#banner.image}}` and `{{#banner.video}}` sections in every template. They are **mutually exclusive** — the system renders only the block matching the actual asset type.

#### Conditional Section Behavior

Given:

```html
{{#banner.image}}
<img src="{{banner.image_url}}" alt="" />
{{/banner.image}} {{#banner.video}}
<video
  src="{{banner.video_url}}"
  poster="{{banner.video_thumbnail_url}}"
></video>
{{/banner.video}}
```

When the banner is an **image**, the output is:

```html
<img src="https://cdn.example.com/banner.jpg" alt="" />
```

The `{{#banner.video}}` block is removed entirely. When the banner is a **video**, only the `{{#banner.video}}` block is rendered.

### Nested Sections and Catalog Item Indexing

Conditional sections may be nested. For example, `{{#brand_logo}}` can appear inside `{{#banner.image}}`.

Catalog items use 0-based indexing (`{{catalog_items[0].*}}`, `{{catalog_items[1].*}}`, etc.). There is no strict upper limit — items beyond the available catalog data are not rendered.

## Rules & Constraints

### Allowed

- Standard HTML tags and attributes (`<div>`, `<img>`, `<video>`, `<p>`, `<span>`, etc.)
- Inline CSS styles and `<style>` blocks
- External resources (images, fonts, CSS from CDNs)
- CSS animations and transitions (`@keyframes`, `transition`, `animation`)

### Forbidden

| Element / Attribute                              | Reason                             |
| ------------------------------------------------ | ---------------------------------- |
| `<script>`                                       | JavaScript execution is blocked    |
| `<iframe>`                                       | Nested frames are not permitted    |
| `<object>`                                       | Embedded objects are not permitted |
| `<embed>`                                        | Plugin content is not permitted    |
| `<form>`                                         | Form submissions are not permitted |
| `on*` event handlers (e.g., `onclick`, `onload`) | Inline JavaScript is blocked       |

### Size Limit

Maximum template file size: **500 KB**.

### Validation

When a template is submitted, the system performs automatic validation:

- **Errors** (must be resolved): forbidden tags detected, missing required macros, size limit exceeded
- **Warnings** (review recommended): unrecognized macros detected

The exact set of required vs. optional macros depends on the platform configuration.

### External Resources

External resources such as CDN-hosted fonts (e.g., Google Fonts), stylesheets, and images are permitted. However, external resources may not load consistently in all preview contexts. Templates should degrade gracefully if external resources are unavailable.

### CSS Animations

CSS `@keyframes`, `transition`, and `animation` properties are permitted. Animations must not depend on JavaScript-triggered class changes, as the sandboxed environment blocks all script execution.

## Ad Type Reference

| Feature        | Sponsored Display  | Sponsored Brands   |
| -------------- | ------------------ | ------------------ |
| Headline       | Supported          | Supported          |
| Custom Text    | Supported          | Supported          |
| CTA            | Supported          | Supported          |
| Banner (Image) | Supported          | Supported          |
| Banner (Video) | Platform-dependent | Platform-dependent |
| Brand Logo     | Supported          | Supported          |
| Catalog Items  | Not available      | Supported          |

## Template Examples

### Sponsored Display (Basic)

```html
<div style="width: 100%; height: 100%; position: relative; overflow: hidden;">
  <h1>{{headline.text}}</h1>

  {{#banner.image}}
  <img src="{{banner.image_url}}" alt="" style="width: 100%; height: auto;" />
  {{/banner.image}} {{#banner.video}}
  <video
    src="{{banner.video_url}}"
    autoplay
    muted
    loop
    playsinline
    poster="{{banner.video_thumbnail_url}}"
    style="width: 100%; height: auto;"
  ></video>
  {{/banner.video}} {{#brand_logo}}
  <img
    src="{{brand_logo.image_url}}"
    alt="Logo"
    style="width: 48px; height: 48px;"
  />
  {{/brand_logo}}

  <button>{{cta.text}}</button>
</div>
```

### Sponsored Brands (with Catalog Items)

```html
<div style="width: 100%; height: 100%; position: relative; overflow: hidden;">
  <h1>{{headline.text}}</h1>

  {{#banner.image}}
  <img src="{{banner.image_url}}" alt="" style="width: 100%;" />
  {{/banner.image}} {{#banner.video}}
  <video
    src="{{banner.video_url}}"
    autoplay
    muted
    loop
    playsinline
    poster="{{banner.video_thumbnail_url}}"
    style="width: 100%;"
  ></video>
  {{/banner.video}} {{#brand_logo}}
  <img src="{{brand_logo.image_url}}" alt="Logo" />
  {{/brand_logo}} {{#catalog_items}}
  <div style="display: flex; gap: 8px;">
    <div>
      <img src="{{catalog_items[0].image_url}}" alt="" />
      <p>{{catalog_items[0].title}}</p>
      <p>{{catalog_items[0].price}}</p>
      <p>{{catalog_items[0].sale_price}}</p>
    </div>
    <div>
      <img src="{{catalog_items[1].image_url}}" alt="" />
      <p>{{catalog_items[1].title}}</p>
      <p>{{catalog_items[1].price}}</p>
      <p>{{catalog_items[1].sale_price}}</p>
    </div>
    <div>
      <img src="{{catalog_items[2].image_url}}" alt="" />
      <p>{{catalog_items[2].title}}</p>
      <p>{{catalog_items[2].price}}</p>
      <p>{{catalog_items[2].sale_price}}</p>
    </div>
  </div>
  {{/catalog_items}}

  <button>{{cta.text}}</button>
</div>
```

## When Generating Templates

1. Set `overflow: hidden` on the root container
2. Use both `{{#banner.image}}` and `{{#banner.video}}` sections — never omit either
3. For video, always include `autoplay muted loop playsinline` and `poster="{{banner.video_thumbnail_url}}"`
4. Wrap catalog items in `{{#catalog_items}}...{{/catalog_items}}`
5. Use 0-based indexing for catalog items: `{{catalog_items[0].*}}`, `{{catalog_items[1].*}}`, etc.
6. Do not use any forbidden tags or `on*` attributes
7. External fonts/CSS are allowed but may not load in all preview contexts — degrade gracefully

## Submission Checklist

- [ ] Template renders correctly at the intended dimensions
- [ ] Both `{{#banner.image}}` and `{{#banner.video}}` sections are included
- [ ] All required macros for the target ad type are present
- [ ] No forbidden elements or `on*` attributes
- [ ] Template file size is under 500 KB
- [ ] Catalog items section is wrapped in `{{#catalog_items}}...{{/catalog_items}}` (Sponsored Brands only)

## FAQ

**Q: Can I use JavaScript?**
A: No. Templates are rendered in a sandboxed environment that blocks all script execution. Use CSS for animations.

**Q: Can I use external fonts or CSS?**
A: Yes, via `<link>` tags or `@import` in a `<style>` block. External resources may not load in all preview contexts.

**Q: What is the maximum number of catalog items?**
A: No strict limit. Use 0-based indexing. The number rendered depends on the advertiser's catalog data.

**Q: How do I submit my template?**
A: Provide the completed HTML file to your account representative. Templates are uploaded via an internal admin tool.

**Q: What happens if I use an unrecognized macro?**
A: The system flags it as a warning during validation. Use only the macros listed in this document.
