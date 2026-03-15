# MOCKUP.html Index Page Template

Template for the clickable sitemap/index page that links to all role screens.
This is a **standalone static HTML file** — it does not use the Node.js server.
Links open role screens in a **new browser tab** via the running Node.js server.

## Placeholders

| Placeholder | Description |
|-------------|-------------|
| `{{APP_NAME}}` | Application name |
| `{{APP_DESCRIPTION}}` | Brief application description from PRD.md context |
| `{{ROLE_SECTIONS}}` | Generated HTML for each role and its screens |
| `{{TOTAL_SCREENS}}` | Total number of screen files |
| `{{DESIGN_PRIMARY}}` | Primary color hex |
| `{{HEADING_FONT}}` | Heading font |
| `{{BODY_FONT}}` | Body font |
| `{{GOOGLE_FONTS_URL}}` | Google Fonts import URL |
| `{{VERSION}}` | Target version if specified, or latest version found in PRD.md |
| `{{VERSION_LABEL}}` | "Generated for v1.0.1" or "Generated for latest (all versions)" |
| `{{FILTER_SUMMARY}}` | Summary of included/excluded items (e.g., "12 items included, 3 excluded") |
| `{{SERVER_PORT}}` | Node.js server port (default: 3000) |

## Template

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>{{APP_NAME}} - UI Mockup Sitemap</title>
  <script src="https://cdn.tailwindcss.com"></script>
  <link rel="preconnect" href="https://fonts.googleapis.com">
  <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
  <link href="{{GOOGLE_FONTS_URL}}" rel="stylesheet">
  <script>
    tailwind.config = {
      theme: {
        extend: {
          colors: {
            primary: '{{DESIGN_PRIMARY}}',
          },
          fontFamily: {
            heading: ['{{HEADING_FONT}}', 'sans-serif'],
            body: ['{{BODY_FONT}}', 'sans-serif'],
          }
        }
      }
    }
  </script>
  <style>
    body { font-family: '{{BODY_FONT}}', sans-serif; }
    h1, h2, h3 { font-family: '{{HEADING_FONT}}', sans-serif; }
  </style>
</head>
<body class="bg-slate-50 min-h-screen">

  <div class="max-w-5xl mx-auto px-6 py-12">

    <!-- Server Notice Banner -->
    <div class="mb-8 bg-amber-50 border border-amber-200 rounded-xl px-5 py-4 flex items-start gap-3">
      <svg class="w-5 h-5 text-amber-500 shrink-0 mt-0.5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
        <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2"
          d="M13 16h-1v-4h-1m1-4h.01M21 12a9 9 0 11-18 0 9 9 0 0118 0z"/>
      </svg>
      <div>
        <p class="text-sm font-semibold text-amber-800">Mockup Server Required</p>
        <p class="text-sm text-amber-700 mt-0.5">
          Screen links open via the local Node.js server at
          <code class="bg-amber-100 px-1.5 py-0.5 rounded text-amber-900 font-mono text-xs">http://localhost:{{SERVER_PORT}}</code>.
          Start it with:
          <code class="bg-amber-100 px-1.5 py-0.5 rounded text-amber-900 font-mono text-xs">cd mockup &amp;&amp; npm install &amp;&amp; npm start</code>
        </p>
      </div>
    </div>

    <!-- Header -->
    <div class="mb-10">
      <div class="flex items-center gap-3 mb-3">
        <div class="w-10 h-10 rounded-xl flex items-center justify-center" style="background-color: {{DESIGN_PRIMARY}}">
          <svg class="w-6 h-6 text-white" fill="none" stroke="currentColor" viewBox="0 0 24 24">
            <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2"
              d="M4 6a2 2 0 012-2h2a2 2 0 012 2v2a2 2 0 01-2 2H6a2 2 0 01-2-2V6zm10 0a2 2 0 012-2h2a2 2 0 012 2v2a2 2 0 01-2 2h-2a2 2 0 01-2-2V6zM4 16a2 2 0 012-2h2a2 2 0 012 2v2a2 2 0 01-2 2H6a2 2 0 01-2-2v-2zm10 0a2 2 0 012-2h2a2 2 0 012 2v2a2 2 0 01-2 2h-2a2 2 0 01-2-2v-2z"/>
          </svg>
        </div>
        <h1 class="text-3xl font-bold text-slate-900">{{APP_NAME}}</h1>
      </div>
      <p class="text-slate-500 text-lg">{{APP_DESCRIPTION}}</p>
      <div class="flex items-center gap-4 mt-4 text-sm text-slate-400">
        <span>{{VERSION}}</span>
        <span class="w-1 h-1 bg-slate-300 rounded-full"></span>
        <span>{{TOTAL_SCREENS}} screens</span>
        <span class="w-1 h-1 bg-slate-300 rounded-full"></span>
        <span>UI Mockup for Designer Review</span>
      </div>
      <div class="flex items-center gap-4 mt-2 text-xs text-slate-400">
        <span>{{VERSION_LABEL}}</span>
        <span class="w-1 h-1 bg-slate-300 rounded-full"></span>
        <span>{{FILTER_SUMMARY}}</span>
      </div>
    </div>

    <!-- Role Sections -->
    {{ROLE_SECTIONS}}

    <!-- Footer -->
    <div class="mt-12 pt-6 border-t border-slate-200 text-center text-sm text-slate-400">
      Generated for UI/UX designer review. Screens open in new tabs via the local mockup server.
    </div>
  </div>

</body>
</html>
```

## Role Section Template

For each role, generate this block inside `{{ROLE_SECTIONS}}`:

```html
<!-- Role: {{ROLE_NAME}} -->
<div class="mb-10">
  <div class="flex items-center gap-2 mb-4">
    <svg class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24" style="color: {{DESIGN_PRIMARY}}">
      <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2"
        d="M16 7a4 4 0 11-8 0 4 4 0 018 0zM12 14a7 7 0 00-7 7h14a7 7 0 00-7-7z"/>
    </svg>
    <h2 class="text-xl font-semibold text-slate-800">{{ROLE_NAME}}</h2>
    <span class="text-sm text-slate-400 ml-2">({{SCREEN_COUNT}} screens)</span>
    <!-- Quick launch link opens the role home in a new tab -->
    <a href="http://localhost:{{SERVER_PORT}}/{{ROLE_FOLDER}}/home"
       target="_blank"
       rel="noopener noreferrer"
       class="ml-auto flex items-center gap-1 text-xs px-3 py-1.5 rounded-lg border transition-colors"
       style="color: {{DESIGN_PRIMARY}}; border-color: {{DESIGN_PRIMARY}}">
      <svg class="w-3.5 h-3.5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
        <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2"
          d="M10 6H6a2 2 0 00-2 2v10a2 2 0 002 2h10a2 2 0 002-2v-4M14 4h6m0 0v6m0-6L10 14"/>
      </svg>
      Open Role Dashboard
    </a>
  </div>
  <div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4">
    {{SCREEN_CARDS}}
  </div>
</div>
```

## Screen Card Template

Each card opens the corresponding screen in a **new browser tab**:

```html
<a href="http://localhost:{{SERVER_PORT}}/{{ROLE_FOLDER}}/{{SCREEN_FILE_NO_EXT}}"
   target="_blank"
   rel="noopener noreferrer"
   class="block bg-white rounded-xl border border-slate-200 p-4 hover:shadow-md transition-all duration-200 cursor-pointer group">
  <div class="flex items-center justify-between mb-2">
    <div class="flex items-center gap-2">
      <svg class="w-4 h-4 text-slate-400 group-hover:text-primary transition-colors"><!-- module icon --></svg>
      <h3 class="font-medium text-slate-800 group-hover:text-primary transition-colors">{{SCREEN_TITLE}}</h3>
    </div>
    <!-- External link icon to indicate opens in new tab -->
    <svg class="w-3.5 h-3.5 text-slate-300 group-hover:text-primary transition-colors shrink-0" fill="none" stroke="currentColor" viewBox="0 0 24 24">
      <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2"
        d="M10 6H6a2 2 0 00-2 2v10a2 2 0 002 2h10a2 2 0 002-2v-4M14 4h6m0 0v6m0-6L10 14"/>
    </svg>
  </div>
  <p class="text-sm text-slate-500">{{SCREEN_DESCRIPTION}}</p>
  <div class="mt-3 flex flex-wrap gap-1">
    <!-- User story tags as badges -->
    <span class="text-xs bg-slate-100 text-slate-500 px-2 py-0.5 rounded-full">{{US_TAG}}</span>
  </div>
</a>
```

The `{{SCREEN_DESCRIPTION}}` is a brief summary of what the screen shows,
derived from the user story action (e.g., "Search and manage geographical information").

The `{{SCREEN_FILE_NO_EXT}}` is the filename without `.html`
(e.g., `location_information` for `location_information.html`).
