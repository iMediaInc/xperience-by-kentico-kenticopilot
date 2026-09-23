# Imgix → image variant mappings

## ImageService API

KX13 `ImgixService` is static. XbyK `ImageService` is scoped with `Instance` for Razor.

```csharp
// KX13
ImgixService.GetImgixUrl(imageUrl, "News Grid Widget.ImgixSettings", isMobile: true)
ImgixService.GetAdHocImgixUrl(imageUrl, leftoverJson)

// XbyK
ImageService.Instance.GetImageUrl(imageUrl, "News Grid Widget.ImgixSettings", isMobile: true)
ImageService.Instance.GetImageUrl(imageUrl, leftoverJson)
```

`ImageUrlService.GetImageUrl` is the implementation. `ImageService` is the only type callers should use after a file is touched.

## Variant sizes (admin + `ImageVariantCatalog`)

Create SmartCrop variants if they're missing. do this by looking through the widgets to find any existing croppings. add them to the imageVariants in XByK and also into the ImageVariantCatalog.


## Widget property template

One leftover field → one dropdown. Keep leftover unannotated.

```csharp
[DropDownComponent(
    Order = 10,
    Label = "Image variant",
    DataProviderType = typeof(ImageVariantOptionsProvider),
    ExplanationText = "Smart-crop size for the image. Automatic keeps the migrated Imgix mapping.")]
public string ImageVariant { get; set; } = string.Empty;

public string ImgixSettings { get; set; } = string.Empty;
```

Resolve in the ViewComponent:

```csharp
var preset = ImageVariantCatalog.Preset(
    properties.ImageVariant,
    properties.ImgixSettings,
    "ImageWidget.ImgixSettings");
```

### Multiple leftovers

| Leftover | Dropdown | Default preset string |
| --- | --- | --- |
| `ImgixSettings` | `ImageVariant` | `{Widget display or class}.ImgixSettings` |
| `ImgixSettings1`…`4` | `ImageVariant1`…`4` | `Feature Card Widget.ImgixSettings1` |
| `ImgixSettingsThumbnail` | `ImageVariantThumbnail` | `Media Gallery Widget.ImgixSettingsThumbnail` |
| `ImgixSettingsFull` | `ImageVariantFull` | `Media Gallery Widget.ImgixSettingsFull` |

Canonical examples: `ImageWidgetProperties`, `NewsGridWidgetProperties`, `MediaGalleryWidgetProperties`, `FeatureCardWidgetProperties`.

## Leftover JSON shapes

`ImageVariantCatalog.NormalizePresetName` accepts:

```json
{ "SettingsName": "News Grid Widget.ImgixSettings" }
{ "WidgetName": "Image Widget", "PropertyName": "ImgixSettings" }
```

Pass the raw leftover string into `Preset(...)` / `GetImageUrl(...)`. Do not parse it in the widget.
