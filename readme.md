Automatically converts image markup to use an edge transformer, and performance optimized layout + CSS logic.

Intercepts various flavors of WordPress' native `wp_get_attachment_image()` and similar, and:
- Uses named (or h/w array value) sizes as lookups for custom behaviour.
- Wraps the `<img>` in a `<picture>` elem.

## Filters

### Enabling/disabling
- `edge_images_disable` (`bool`): Disable all image transformation mechanisms. Defaults to `false`.
- `edge_images_exclude` (`array`): An array of images to exclude from transformation.
- `edge_images_force_transform` (`bool`): Forcibly enable transformation, even if environmental settings would otherwise disable it (e.g., if a site is in a local environment). Defaults to `false`.

### General configuration
- `edge_images_provider` (`str`): The name of the edge provider to use. Defaults to `Cloudflare`.
- `edge_images_domain` (`str`): The fully qualified domain name (and protocol) to use to as the base for image transformation. Defaults to `get_site_url()`.
- `edge_images_content_width` (`int`): The default maximum content width for an image. Defaults to the theme's `$content_width` value, or falls back to `600`.

### Image quality settings
- `edge_images_quality_low` (`int`): The value to use for low quality images (from `1`-`100`). Defaults to `65`.
- `edge_images_quality_medium` (`int`): The value to use for low quality images (from `1`-`100`). Defaults to `75`.
- `edge_images_quality_high` (`int`): The value to use for low quality images (from `1`-`100`). Defaults to `85`.

### `srcset` generation settings
- `edge_images_step_value` (`int`): The number of pixels to increment in `srcset` variations. Defaults to `100`.
- `edge_images_min_width` (`int`): The minimum width to generate in an `srcset`. Defaults to `400`.
- `edge_images_max_width` (`int`): The maximum width to generate in an `srcset`. Defaults to `2400`.

### Specifying image sizes
The `edge_images_sizes` filter is used to define an associative array of named image sizes.
These accept the following properties.

#### Required
- `height` (`int`): The height in pixels of the image of the smallest/mobile/default size. Sets the `height` attribute on the `<img>` elem.
- `width` (`int`): The `width` in pixels of the image of the smallest/mobile/default size. Sets the `width` attribute on the `<img>` elem.

#### Optional
- `sizes` (`str`):  The `sizes` attribute to be used on the `<img>` elem.
- `srcset` (`arr`): An array of `width`/`height` arrays. Used to generate the `srcset` attribute (and stepped variations) on the `<img>` elem.
- `fit` (`str`): Sets the `fit` attribute on the `<img>` elem. Defaults to `cover`.
  - Options differ by edge providers (see https://developers.cloudflare.com/images/image-resizing/url-format).
- `layout` (`str`): Determines how `<img>` markup should be generated, based on whether the image is `responsive` or has `fixed` dimensions. Defaults to `responsive`.
- `loading` (`str`): Sets the `loading` attribute on the `<img>` elem. Defaults to `lazy`.
- `decoding` (`str`): Sets the `decoding` attribute on the `<img>` elem. Defaults to `async`.
- `class` (`array`|`str`): Extends the `class` value(s) on the `<img>` elem.
  - Always outputs `attachment-%size% size-%size% edge-images-img` (where `%size%` is the sanitized image size name).
- `picture-class` (`array`|`str`): Extends the `class` value(s) on the `<picture>` elem.
  - Always outputs `layout-%layout% picture-%size% edge-images-picture image-id-%id%` (where `%size%` is the sanitized image size name, `%layout%` is the `layout` value, and `%id%` is the attachment ID).

#### Example configurations:
A general use-case, which defines dimensions, sizes, and custom `srcset` values.
```
$sizes['example_size_1'] = array(
  'width'   => 173,
  'height'  => 229,
  'sizes'   => '(max-width: 768px) 256px, 173px',
  'srcset'  => array(
    array(
      'width'  => 256,
      'height' => 229,
    )
  )
);
```

A simple small image.
```
$sizes['small_logo'] = array(
  'width'  => 70,
  'height' => 20,
  'sizes'  => '70px'
);
```

A simple small image, requested with a size array (of `[32, 32]`) instead of a named size.
```
$sizes['32x32'] = array(
  'width'  => 32,
  'height' => 32,
  'sizes'  => '32px'
);
```

A more complex use-case, which changes layout considerably at different viewport ranges (and has complex `sizes` and `srcset` values to support this).
```
$sizes['card'] = array(
  'width'  => 195,
  'height' => 195,
  'sizes'  => '(max-width: 1120px) 25vw, (min-width: 1121px) and (max-width: 1440px) 150px, 195px',
  'srcset' => array(
    array(
      'width'  => 150,
      'height' => 150,
    ),
    array(
      'width'  => 125,
      'height' => 125,
    ),
    array(
      'width'  => 100,
      'height' => 100,
    ),
  ),
  'loading' => 'eager',
  'picture-class' => 'pineapple pizza'
);

```

## Example outputs
Using `wp_get_attachment_image( $image_id, 'banner' );`

**Before**:
```
<img
  width="1920"
  height="580"
  src="https://www.example.com/path-to-image-1920x580.jpg"
  class="attachment-banner size-banner"
  alt=""
  loading="lazy">
```

**After**:

With the addition of the following filter:
```
public function register_edge_image_sizes( array $sizes ) : array {
  $sizes['banner'] = array(
    'width'   => 968,
    'height'  => 500,
    'sizes'   => '(max-width: 968px) calc(100vw - 2.5em), 968px',
    'loading' => 'eager',
  );
}
add_filter( 'edge_images_sizes', array( $instance, 'register_edge_image_sizes' ), 1, 1 );

```

```
<picture
  style="--aspect-ratio:968/500"
  class="picture-banner edge-images-picture responsive image-id-34376">
<img
  class="attachment-banner size-banner edge-images-img"
  alt=""
  decoding="async"
  height="500"
  loading="eager"
  sizes="(max-width: 968px) calc(100vw - 2.5em), 968px"
  src="https://www.example.com/cdn-cgi/image/f=auto%2Cfit=cover%2Cgravity=auto%2Cheight=500%2Cmetadata=none%2Conerror=redirect%2Cq=85%2Cwidth=968/path-to-image.jpg" width="968"
  srcset="https://www.example.com/cdn-cgi/image/f=auto%2Cfit=cover%2Cgravity=auto%2Cheight=750%2Cmetadata=none%2Conerror=redirect%2Cq=75%2Cwidth=1452/path-to-image.jpg 1452w,
  	  https://www.example.com/cdn-cgi/image/f=auto%2Cfit=cover%2Cgravity=auto%2Cheight=1000%2Cmetadata=none%2Conerror=redirect%2Cq=65%2Cwidth=1936/path-to-image.jpg 1936w,
  	  https://www.example.com/cdn-cgi/image/f=auto%2Cfit=cover%2Cgravity=auto%2Cheight=207%2Cmetadata=none%2Conerror=redirect%2Cq=85%2Cwidth=400/path-to-image.jpg 400w,
  	  https://www.example.com/cdn-cgi/image/f=auto%2Cfit=cover%2Cgravity=auto%2Cheight=259%2Cmetadata=none%2Conerror=redirect%2Cq=85%2Cwidth=500/path-to-image.jpg 500w,
          https://www.example.com/cdn-cgi/image/f=auto%2Cfit=cover%2Cgravity=auto%2Cheight=310%2Cmetadata=none%2Conerror=redirect%2Cq=85%2Cwidth=600/path-to-image.jpg 600w,
	  https://www.example.com/cdn-cgi/image/f=auto%2Cfit=cover%2Cgravity=auto%2Cheight=362%2Cmetadata=none%2Conerror=redirect%2Cq=85%2Cwidth=700/path-to-image.jpg 700w,
	  https://www.example.com/cdn-cgi/image/f=auto%2Cfit=cover%2Cgravity=auto%2Cheight=414%2Cmetadata=none%2Conerror=redirect%2Cq=85%2Cwidth=800/path-to-image.jpg 800w,
	  https://www.example.com/cdn-cgi/image/f=auto%2Cfit=cover%2Cgravity=auto%2Cheight=465%2Cmetadata=none%2Conerror=redirect%2Cq=85%2Cwidth=900/path-to-image.jpg 900w,
	  https://www.example.com/cdn-cgi/image/f=auto%2Cfit=cover%2Cgravity=auto%2Cheight=517%2Cmetadata=none%2Conerror=redirect%2Cq=85%2Cwidth=1000/path-to-image.jpg 1000w,
	  https://www.example.com/cdn-cgi/image/f=auto%2Cfit=cover%2Cgravity=auto%2Cheight=620%2Cmetadata=none%2Conerror=redirect%2Cq=85%2Cwidth=1200/path-to-image.jpg 1200w,
	  https://www.example.com/cdn-cgi/image/f=auto%2Cfit=cover%2Cgravity=auto%2Cheight=724%2Cmetadata=none%2Conerror=redirect%2Cq=85%2Cwidth=1400/path-to-image.jpg 1400w,
	  https://www.example.com/cdn-cgi/image/f=auto%2Cfit=cover%2Cgravity=auto%2Cheight=827%2Cmetadata=none%2Conerror=redirect%2Cq=85%2Cwidth=1600/path-to-image.jpg 1600w,
	  https://www.example.com/cdn-cgi/image/f=auto%2Cfit=cover%2Cgravity=auto%2Cheight=930%2Cmetadata=none%2Conerror=redirect%2Cq=85%2Cwidth=1800/path-to-image.jpg 1800w,
	  https://www.example.com/cdn-cgi/image/f=auto%2Cfit=cover%2Cgravity=auto%2Cheight=500%2Cmetadata=none%2Conerror=redirect%2Cq=85%2Cwidth=968/path-to-image.jpg 968w,
	  https://www.example.com/cdn-cgi/image/f=auto%2Cfit=cover%2Cgravity=auto%2Cheight=250%2Cmetadata=none%2Conerror=redirect%2Cq=85%2Cwidth=484/path-to-image.jpg 484w">
</picture>
```
