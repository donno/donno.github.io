---
name: SVG to PNG
---

This uses [Inkscape](0) rather than [Imagemagic](1) as the later had trouble with
the gradients in SVGs.

```
find /icons -type f -iname '*.svg' \
  -exec sh -c 'for f do
    inkscape "$f" \
      --export-type=png \
      --export-width=128 \
      --export-filename="${f%.*}.png"


    convert "${f%.*}.png" \
      -background none \
      -gravity center \
      -resize 128x128 \
      -extent 128x128 \
      "${f%.*}.png"
  done' sh {} +
```

## Container Set-up
```
podman run -it --rm -v .:/icons public.ecr.aws/ubuntu/ubuntu:26.04
# Within the container
apt-get update && (DEBIAN_FRONTEND=noninteractive apt-get install inkscape)
```

[0]: https://gitlab.com/inkscape/inkscape
[1]: https://github.com/ImageMagick/ImageMagick