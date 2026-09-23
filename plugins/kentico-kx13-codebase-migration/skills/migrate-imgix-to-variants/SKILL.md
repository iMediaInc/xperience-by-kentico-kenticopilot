---
name: migrate-imgix-to-variants
description: >-
  Migrates leftover KX13 ImgixSettings to XbyK smart-crop image variants and
  routes ImgixService calls through ImageService. Use when the user asks to
  migrate Imgix, ImgixSettings, ImgixService, image variants, ImageService, or
  replace imgix.net URLs with /getContentAsset variants.
argument-hint: "[widget-or-call-site]"
---
# Migrate Imgix settings to image variants
Replace KX13 `ImgixService` / leftover `ImgixSettings` with XbyK **image variants**. Every former Imgix URL must go through **`ImageService`**, which resolves a variant via `ImageVariantCatalog` and `ImageUrlService`. Do not emit `imgix.net` query strings.
API mapping, catalog keys, and the widget property template: [mappings.md](mappings.md).
## Hard rules
- Never call `*.ImgixService` from XbyK. Do not port `FlushCache` or imgix purge.
- Do not rebuild `w=` / `h=` / `fit=` query strings. Variants are smart-crop binaries on `/getContentAsset/{item}/{field}/{variant}/{file}_{variant}.ext`.
- Leftover Imgix JSON/string properties stay **unannotated** (no `[LeftoverLifted]`, no admin component). Keep the property so remigrated Page Builder JSON still binds.
- Do not revert leftover lift or Content Hub selectors while touching a widget.
- Isolated build: `dotnet build src/xbyk/*.csproj -o src/xbyk/obj/verify-build`. Restart after ViewComponent / Razor / widget property changes.
## Existing pieces (extend these)
| Piece | Path |
| --- | --- |
| Public facade (create if missing) | `src/*.Business/Services/ImageService.cs` (`IImageService`) |
| Asset + variant URL resolver | `src/*.Business/Services/ImageUrlService.cs` |
| Preset → variant map | `src/*.Business/Imaging/ImageVariantCatalog.cs` |
| `/getContentAsset` rewriter | `src/*.Business/Imaging/ImageVariantUrl.cs` |
| Admin dropdown | `src/xbyk/Helpers/ImageVariantOptionsProvider.cs` |
| DI | `src/xbyk/Helpers/ServiceCollectionExtensions.cs` — scoped `ImageUrlService`; add scoped `IImageService` / `ImageService` |
| Canonical widget | `src/xbyk/Components/Widgets/Misc/ImageWidget/` |
`ImageService` **delegates** to `ImageUrlService`. Do not add a second Content Hub lookup.
## Workflow
Copy and track:
```
Imgix → variants:
- [ ] Ensure ImageService exists and is registered
- [ ] Inventory ImgixService / leftover ImgixSettings / ImageUrlService.GetImageUrl
- [ ] Add catalog entries for any unmapped preset
- [ ] Add ImageVariant dropdown + resolve via ImageVariantCatalog.Preset
- [ ] Route URL generation through ImageService
- [ ] Isolated build + restart
```
If the user named a widget or file, do only that surface. If they asked for a full pass, inventory first, then migrate one call-site group at a time.
### 1. Ensure ImageService
If `IImageService` / `ImageService` is missing, create it in `*.Business.Services`:
```csharp
public interface IImageService
{
    HtmlString GetImageUrl(string? source, string? presetName = null, bool isMobile = false);
}
public class ImageService : IImageService
{
    public static IImageService Instance =>
        _instance ?? throw new InvalidOperationException("ImageService has not been initialized.");
    public HtmlString GetImageUrl(string? source, string? presetName = null, bool isMobile = false)
        => ImageUrlService.Instance.GetImageUrl(source, presetName, isMobile);
}
```
Register scoped next to `ImageUrlService`. Set `Instance` in the constructor the same way `ImageUrlService` does. Warm it in `Program.cs` the same way (`GetService<IImageService>()` or `GetService<ImageService>()`).
Method mapping from KX13:
| KX13 | XbyK |
| --- | --- |
| `ImgixService.GetImgixUrl(url, preset, override, isMobile)` | `ImageService.Instance.GetImageUrl(url, preset, isMobile)` — ignore settings overrides |
| `ImgixService.GetAdHocImgixUrl(url, settingsJson, override)` | `GetImageUrl(url, settingsJson)` — catalog parses `{ SettingsName }` / `{ WidgetName, PropertyName }` |
| `ImgixService.FlushCache` | Drop |
`presetName` may be a leftover JSON blob, a KX13 settings name (`News Grid Widget.ImgixSettings`), or a variant code (`Card`). `ImageVariantCatalog.Resolve` already handles all three.
### 2. Inventory
Grep XbyK (`src/xbyk`, `src/.Business`) for:
- `ImgixService.`
- `GetImgixUrl` / `GetAdHocImgixUrl`
- `ImgixSettings` on `*Properties.cs`
- `ImageUrlService.Instance.GetImageUrl` (switch these to `ImageService` when you touch the file)
Grep KX13 `kx13/` for `ImgixService.` to catch call sites not ported yet.
### 3. Catalog
For each leftover preset string, add a `PresetToVariant` entry in `ImageVariantCatalog` if missing. Keys are case-insensitive. Normalize like the catalog: leftover JSON `SettingsName`, or `{Widget}.{Property}`.
Pick the variant from [mappings.md](mappings.md) sizes. If a widget uses a new crop, add a code name to `AllVariantCodeNames` **and** tell the user to create the matching smart-crop variant under Content types → Asset configurations → Image variants. Do not invent a variant that is not in admin.
### 4. Widget leftover ImgixSettings → ImageVariant
On the widget (or other properties type):
1. Add `ImageVariant` (`string`) with `[DropDownComponent]` + `ImageVariantOptionsProvider`. Label: `Image variant`. Explanation: `Smart-crop size. Automatic keeps the migrated Imgix mapping.`
2. Keep leftover `ImgixSettings` / `ImgixSettings1` / `ImgixSettingsThumbnail` as a bare `string` (unannotated).
3. In the ViewComponent, resolve once:
```csharp
var preset = ImageVariantCatalog.Preset(
    properties.ImageVariant,
    properties.ImgixSettings,
    "ImageWidget.ImgixSettings");
```
`Preset` prefers the dropdown, then leftover JSON/name, then the default KX13 settings name.
4. Generate URLs only through `ImageService`:
```csharp
var url = ImageService.Instance.GetImageUrl(rawUrl, preset)?.ToString() ?? rawUrl;
var mobile = ImageService.Instance.GetImageUrl(rawUrl, preset, isMobile: true)?.ToString() ?? rawUrl;
```
Do not pass leftover JSON into the view as an imgix query. Pass the **resolved preset or already-built URL**.
Multiple leftover fields (thumbnail/full, image1/image2): one `ImageVariant*` dropdown per leftover field. See Media Gallery / Feature Cards in [mappings.md](mappings.md).
### 5. Non-widget call sites
Views, metadata, account partials, Tessitura sync: replace `ImgixService.GetImgixUrl(...)` / `ImageUrlService.Instance.GetImageUrl(...)` with `ImageService.Instance.GetImageUrl(...)`. Keep the same preset string so the catalog mapping still hits. Use `isMobile: true` on the mobile `<img>` only.
### 6. Verify
- Isolated build (path above).
- Restart. Confirm admin shows **Image variant** (Automatic + named sizes) and does **not** show leftover Imgix as an editor.
- One desktop and one mobile `<img>`: `src` is `/getContentAsset/.../{Variant}/..._{Variant}.ext` (or primary URL when variant is Default / empty), never `*.imgix.net`.
- Empty image source still renders empty `src`, not a thrown exception.
- Remigrated widget JSON that only has `ImgixSettings` still resolves via leftover + catalog when dropdown is Automatic.
## Out of this skill’s job
- Legacy media 301s (`legacy-media-redirect`) — those stay on the **primary** asset URL, not a named variant
- Changing leftover Imgix property names (Page Builder JSON keys)
- Enabling or configuring the Imgix CDN
- Pixel-diff of the page (`migrate-code-page-visual`)