<!--
author:   André Dietrich
version:  0.0.2
language: en
narrator: US English Female
logo:     https://upload.wikimedia.org/wikipedia/commons/thumb/3/3d/Wikimedia_Brand_Guidelines_Update_2022_-_Wikimedia_Logo_Main.png/960px-Wikimedia_Brand_Guidelines_Update_2022_-_Wikimedia_Logo_Main.png
comment:  LiaScript template for embedding Wikimedia Commons images with
          auto-fetched metadata (title, caption, artist, license, source).
          Supports single images, individual metadata fields, and galleries.
          Accepts both `File:` identifiers and full Wikimedia Commons URLs.

@onload
window._wikimediaCache = window._wikimediaCache || {};

window._fetchCommonsData = function(fileTitle, width) {
  // Accept full Wikimedia/Wikipedia URLs (any language) and bare namespace identifiers.
  // Extracts "Namespace:Filename" from the URL, then normalises the prefix to "File:".
  const urlMatch = fileTitle.match(/\/wiki\/([^?#]+:[^?#]+)/i);
  if (urlMatch) fileTitle = decodeURIComponent(urlMatch[1]);
  fileTitle = fileTitle.replace(/^[^:]+:/, "File:");

  if (window._wikimediaCache[fileTitle]) {
    return window._wikimediaCache[fileTitle];
  }

  const endpoint = "https://commons.wikimedia.org/w/api.php";

  function cleanText(value) {
    if (!value) return "";
    const div = document.createElement("div");
    div.innerHTML = value;
    return div.textContent.replace(/\s+/g, " ").trim();
  }

  const promise = (async () => {
    const params = new URLSearchParams({
      action: "query",
      format: "json",
      origin: "*",
      prop: "imageinfo",
      titles: fileTitle,
      iiprop: "url|extmetadata|mime|mediatype|size",
      iiurlwidth: String(width || window.innerWidth)
    });

    const res = await fetch(`${endpoint}?${params}`);
    if (!res.ok) throw new Error(`HTTP Fehler: ${res.status}`);

    const data = await res.json();
    const page = Object.values(data.query.pages)[0];

    if (!page || page.missing) throw new Error(`Datei nicht gefunden: ${fileTitle}`);

    const info = page.imageinfo?.[0];
    if (!info) throw new Error(`Keine Bildinformationen gefunden: ${fileTitle}`);

    const meta = info.extmetadata || {};
    const fileName = page.title.replace(/^File:/, "");
    const imageTitle = cleanText(meta.ObjectName?.value) || fileName;

    const mediaType = String(info.mediatype || "").toUpperCase();
    const mime      = String(info.mime || "").toLowerCase();
    let kind;
    if      (mediaType === "AUDIO" || mime.startsWith("audio/"))                               kind = "audio";
    else if (mediaType === "VIDEO" || mime.startsWith("video/"))                               kind = "video";
    else if (mediaType === "BITMAP" || mediaType === "DRAWING" || mime.startsWith("image/"))   kind = "image";
    else if (mediaType === "3D"    || mime.startsWith("model/"))                               kind = "3d";
    else if (mediaType === "OFFICE" || mime === "application/pdf")                             kind = "document";
    else                                                                                       kind = "other";

    return {
      imageUrl: info.thumburl || info.url,
      fileUrl:  info.url,
      source:   info.descriptionurl || "",
      title:    imageTitle,
      caption:  cleanText(meta.ImageDescription?.value) || imageTitle,
      artist:   cleanText(meta.Attribution?.value) || cleanText(meta.Artist?.value) || "",
      license:  cleanText(meta.LicenseShortName?.value) || cleanText(meta.UsageTerms?.value) || "",
      licenseUrl: meta.LicenseUrl?.value || "",
      credit:   cleanText(meta.Credit?.value) || "",
      mime,
      mediaType,
      kind
    };
  })();

  window._wikimediaCache[fileTitle] = promise;
  return promise;
};

window.getCommonsEmbedMarkdown = async function(fileTitle, options = {}) {
  function escapeMarkdownAlt(value) {
    return (value || "").replace(/\\/g, "\\\\").replace(/\]/g, "\\]").replace(/\n/g, " ");
  }
  function escapeMarkdownTitle(value) {
    return (value || "").replace(/\\/g, "\\\\").replace(/"/g, '\\"').replace(/\n/g, " ");
  }
  function escapeMarkdownUrl(value) {
    return (value || "").replace(/\(/g, "%28").replace(/\)/g, "%29").replace(/ /g, "%20");
  }

  const { width = window.innerWidth } = options;
  const d = await window._fetchCommonsData(fileTitle, width);

  const titleText = [
    d.title,
    d.artist   && `by ${d.artist}`,
    d.license  && `License: ${d.license}`,
    d.licenseUrl && `License URL: ${d.licenseUrl}`,
    d.source   && `Source: ${d.source}`
  ].filter(Boolean).join(" · ");

  const alt  = escapeMarkdownAlt(d.caption);
  const url  = escapeMarkdownUrl(d.kind === "audio" || d.kind === "video" ? d.fileUrl : d.imageUrl);
  const meta = escapeMarkdownTitle(titleText);

  if (d.kind === "audio") return `?[${alt}](${url} "${meta}")`;
  if (d.kind === "video") return `!?[${alt}](${url} "${meta}")`;
  return                         `![${alt}](${url} "${meta}")`;
};
@end


@Wikimedia.embed: <script run-once modify="false">
  window.getCommonsEmbedMarkdown("@0")
    .then(markdown => {
      send.lia("LIASCRIPT: " + markdown);
    })
    .catch(error => {
      console.error("Fehler beim Laden der Bilddaten:", error);
    });

  "LIA: wait"
  </script>


@Wikimedia._field: <script run-once modify="false">
  window._fetchCommonsData("@0")
    .then(data => {
      send.lia(data["@1"] || "");
    })
    .catch(error => {
      console.error("Fehler beim Laden der Bilddaten:", error);
    });

  "LIA: wait"
  </script>


@Wikimedia.title: @Wikimedia._field(@0,title)

@Wikimedia.caption: @Wikimedia._field(@0,caption)

@Wikimedia.artist: @Wikimedia._field(@0,artist)

@Wikimedia.license: @Wikimedia._field(@0,license)

@Wikimedia.source: @Wikimedia._field(@0,source)

@Wikimedia.credit: @Wikimedia._field(@0,credit)


@Wikimedia.gallery: <script run-once modify="false">
  Promise.all(
    "@0".split("|").map(f => window.getCommonsEmbedMarkdown(f.trim()))
  ).then(images => {
    send.lia("LIASCRIPT:\n" + images.join("\n"));
  }).catch(error => {
    console.error("Fehler beim Laden der Galerie:", error);
  });

  "LIA: wait"
  </script>
-->

# Wikimedia Commons – LiaScript Template

                          --{{0}}--
This template lets you embed images from
[Wikimedia Commons](https://commons.wikimedia.org) directly into your
[LiaScript](https://liascript.github.io) course — no manual copy-pasting of
URLs or license texts required. Metadata (title, caption, artist, license,
source) is fetched automatically from the Commons API. Responses are cached, so
each file is only loaded once even if referenced by multiple macros.

__Try it on LiaScript:__

https://liascript.github.io/course/?https://raw.githubusercontent.com/LiaTemplates/Wikimedia/main/README.md

__See the project on GitHub:__

https://github.com/LiaTemplates/Wikimedia

                         --{{1}}--
There are three ways to import this functionality into your course, we always recommend option 2.

                           {{1}}
1. Import the latest version _(may include breaking changes)_

   `import: https://raw.githubusercontent.com/LiaTemplates/Wikimedia/main/README.md`

2. Pin a specific release

   `import: https://raw.githubusercontent.com/LiaTemplates/Wikimedia/0.0.2/README.md`

3. Copy the macro definitions from the [Implementation](#implementation) section
   directly into your document header.


## `@Wikimedia.embed`

                          --{{0}}--
Embeds a single media file at the current viewport width. The macro
auto-detects the media type via the Commons API and produces the correct
LiaScript syntax — `![…](…)` for images, `?[…](…)` for audio, and
`!?[…](…)` for video. Alt text, title, artist, license, and source URL
are baked in automatically.
Pass either a `File:` identifier or a full Wikimedia Commons URL.

`@Wikimedia.embed(File:Hoverfly_May_2008-8.jpg)`

@Wikimedia.embed(File:Hoverfly_May_2008-8.jpg)

---

`@Wikimedia.embed(https://commons.wikimedia.org/wiki/File:Turdus_merula_2.ogg)`

@Wikimedia.embed(https://commons.wikimedia.org/wiki/File:Turdus_merula_2.ogg)

---

`@Wikimedia.embed(https://commons.wikimedia.org/wiki/File:Turning_a_Resource_into_an_Open_Educational_Resource.webm)`

@Wikimedia.embed(https://commons.wikimedia.org/wiki/File:Turning_a_Resource_into_an_Open_Educational_Resource.webm)

## Metadata Macros

                          --{{0}}--
Each metadata field has its own macro. All macros share the same internal
cache, so if a file has already been fetched (e.g. by `@Wikimedia.embed`),
no additional network request is made.

| Macro                | Returns                       |
|----------------------|-------------------------------|
| `@Wikimedia.title`   | Object name / filename        |
| `@Wikimedia.caption` | Image description             |
| `@Wikimedia.artist`  | Author / attribution          |
| `@Wikimedia.license` | Short license name            |
| `@Wikimedia.source`  | Link to the Commons file page |
| `@Wikimedia.credit`  | Credit line                   |

---

**Titel:** @Wikimedia.title(File:Zunftwappen.svg)

**Untertitel:** @Wikimedia.caption(File:Zunftwappen.svg)

**Urheber:** @Wikimedia.artist(File:Zunftwappen.svg)

**Lizenz:** @Wikimedia.license(File:Zunftwappen.svg)

**Quelle:** @Wikimedia.source(File:Zunftwappen.svg)

**Credit:** @Wikimedia.credit(File:Zunftwappen.svg)

                          --{{1}}--
All six macros accept the same input formats — `File:` identifier or full URL.

    {{1}}
``` markdown
**Titel:** @Wikimedia.title(File:Zunftwappen.svg)

**Untertitel:** @Wikimedia.caption(File:Zunftwappen.svg)

**Urheber:** @Wikimedia.artist(File:Zunftwappen.svg)

**Lizenz:** @Wikimedia.license(File:Zunftwappen.svg)

**Quelle:** @Wikimedia.source(File:Zunftwappen.svg)

**Credit:** @Wikimedia.credit(File:Zunftwappen.svg)
```


## `@Wikimedia.gallery`

                          --{{0}}--
Embeds multiple images as a LiaScript gallery. Pass a `|`-separated list of
file identifiers or URLs. All fetches run in parallel; each unique file is
still only requested once.

`@Wikimedia.gallery(File:Turdus_merula_2.ogg|File:Turning_a_Resource_into_an_Open_Educational_Resource.webm|Datei:Global_Open_Educational_Resources_Logo.svg)`

@Wikimedia.gallery(File:Turdus_merula_2.ogg|File:Turning_a_Resource_into_an_Open_Educational_Resource.webm|Datei:Global_Open_Educational_Resources_Logo.svg)


## Implementation

                          --{{0}}--
The block below contains the complete macro definitions. Copy it into the
header comment of your LiaScript document to use the template without the
`import` statement.

``` markdown
@onload
window._wikimediaCache = window._wikimediaCache || {};

window._fetchCommonsData = function(fileTitle, width) {
  // Accept full Wikimedia/Wikipedia URLs (any language) and bare namespace identifiers.
  // Extracts "Namespace:Filename" from the URL, then normalises the prefix to "File:".
  const urlMatch = fileTitle.match(/\/wiki\/([^?#]+:[^?#]+)/i);
  if (urlMatch) fileTitle = decodeURIComponent(urlMatch[1]);
  fileTitle = fileTitle.replace(/^[^:]+:/, "File:");

  if (window._wikimediaCache[fileTitle]) {
    return window._wikimediaCache[fileTitle];
  }

  const endpoint = "https://commons.wikimedia.org/w/api.php";

  function cleanText(value) {
    if (!value) return "";
    const div = document.createElement("div");
    div.innerHTML = value;
    return div.textContent.replace(/\s+/g, " ").trim();
  }

  const promise = (async () => {
    const params = new URLSearchParams({
      action: "query",
      format: "json",
      origin: "*",
      prop: "imageinfo",
      titles: fileTitle,
      iiprop: "url|extmetadata|mime|mediatype|size",
      iiurlwidth: String(width || window.innerWidth)
    });

    const res = await fetch(`${endpoint}?${params}`);
    if (!res.ok) throw new Error(`HTTP Fehler: ${res.status}`);

    const data = await res.json();
    const page = Object.values(data.query.pages)[0];

    if (!page || page.missing) throw new Error(`Datei nicht gefunden: ${fileTitle}`);

    const info = page.imageinfo?.[0];
    if (!info) throw new Error(`Keine Bildinformationen gefunden: ${fileTitle}`);

    const meta = info.extmetadata || {};
    const fileName = page.title.replace(/^File:/, "");
    const imageTitle = cleanText(meta.ObjectName?.value) || fileName;

    const mediaType = String(info.mediatype || "").toUpperCase();
    const mime      = String(info.mime || "").toLowerCase();
    let kind;
    if      (mediaType === "AUDIO" || mime.startsWith("audio/"))                               kind = "audio";
    else if (mediaType === "VIDEO" || mime.startsWith("video/"))                               kind = "video";
    else if (mediaType === "BITMAP" || mediaType === "DRAWING" || mime.startsWith("image/"))   kind = "image";
    else if (mediaType === "3D"    || mime.startsWith("model/"))                               kind = "3d";
    else if (mediaType === "OFFICE" || mime === "application/pdf")                             kind = "document";
    else                                                                                       kind = "other";

    return {
      imageUrl: info.thumburl || info.url,
      fileUrl:  info.url,
      source:   info.descriptionurl || "",
      title:    imageTitle,
      caption:  cleanText(meta.ImageDescription?.value) || imageTitle,
      artist:   cleanText(meta.Attribution?.value) || cleanText(meta.Artist?.value) || "",
      license:  cleanText(meta.LicenseShortName?.value) || cleanText(meta.UsageTerms?.value) || "",
      licenseUrl: meta.LicenseUrl?.value || "",
      credit:   cleanText(meta.Credit?.value) || "",
      mime,
      mediaType,
      kind
    };
  })();

  window._wikimediaCache[fileTitle] = promise;
  return promise;
};

window.getCommonsEmbedMarkdown = async function(fileTitle, options = {}) {
  function escapeMarkdownAlt(value) {
    return (value || "").replace(/\\/g, "\\\\").replace(/\]/g, "\\]").replace(/\n/g, " ");
  }
  function escapeMarkdownTitle(value) {
    return (value || "").replace(/\\/g, "\\\\").replace(/"/g, '\\"').replace(/\n/g, " ");
  }
  function escapeMarkdownUrl(value) {
    return (value || "").replace(/\(/g, "%28").replace(/\)/g, "%29").replace(/ /g, "%20");
  }

  const { width = window.innerWidth } = options;
  const d = await window._fetchCommonsData(fileTitle, width);

  const titleText = [
    d.title,
    d.artist   && `by ${d.artist}`,
    d.license  && `License: ${d.license}`,
    d.licenseUrl && `License URL: ${d.licenseUrl}`,
    d.source   && `Source: ${d.source}`
  ].filter(Boolean).join(" · ");

  const alt  = escapeMarkdownAlt(d.caption);
  const url  = escapeMarkdownUrl(d.kind === "audio" || d.kind === "video" ? d.fileUrl : d.imageUrl);
  const meta = escapeMarkdownTitle(titleText);

  if (d.kind === "audio") return `?[${alt}](${url} "${meta}")`;
  if (d.kind === "video") return `!?[${alt}](${url} "${meta}")`;
  return                         `![${alt}](${url} "${meta}")`;
};
@end


@Wikimedia.embed: <script run-once modify="false">
  window.getCommonsEmbedMarkdown("@0")
    .then(markdown => {
      send.lia("LIASCRIPT: " + markdown);
    })
    .catch(error => {
      console.error("Fehler beim Laden der Bilddaten:", error);
    });

  "LIA: wait"
  </script>


@Wikimedia._field: <script run-once modify="false">
  window._fetchCommonsData("@0")
    .then(data => {
      send.lia(data["@1"] || "");
    })
    .catch(error => {
      console.error("Fehler beim Laden der Bilddaten:", error);
    });

  "LIA: wait"
  </script>


@Wikimedia.title: @Wikimedia._field(@0,title)

@Wikimedia.caption: @Wikimedia._field(@0,caption)

@Wikimedia.artist: @Wikimedia._field(@0,artist)

@Wikimedia.license: @Wikimedia._field(@0,license)

@Wikimedia.source: @Wikimedia._field(@0,source)

@Wikimedia.credit: @Wikimedia._field(@0,credit)


@Wikimedia.gallery: <script run-once modify="false">
  Promise.all(
    "@0".split("|").map(f => window.getCommonsEmbedMarkdown(f.trim()))
  ).then(images => {
    send.lia("LIASCRIPT:\n" + images.join("\n"));
  }).catch(error => {
    console.error("Fehler beim Laden der Galerie:", error);
  });

  "LIA: wait"
  </script>
```

                          --{{1}}--
Or paste the raw definitions directly — useful when you want to keep your
course fully self-contained or offline-capable.

    {{1}}
https://raw.githubusercontent.com/LiaTemplates/Wikimedia/main/README.md
