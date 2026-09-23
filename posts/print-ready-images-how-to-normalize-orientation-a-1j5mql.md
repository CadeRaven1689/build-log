# Print-Ready Images: How to Normalize Orientation and Convert in 3 Steps

To prepare print-ready images with an API, normalize orientation and convert to the printer's accepted format when each image enters the system, not when an order is released. The deciding constraint is detection timing: a sideways or wrong-format file found at the printer has already crossed every cheap validation boundary.

**Short answer:** read metadata first, apply the camera rotation, then encode one production derivative while retaining the original privately. For a customer-support workflow that turns prompts into short promo videos and matching print assets, put moderation before that derivative becomes eligible for production. Treat moderation coverage as a release gate, not a checkbox attached to the image library.

This is a three-step ingest decision. It also gives an eval harness a stable target: one source file should yield one known orientation, one declared output format, and one moderation disposition before any order-time code sees it.

## How should an API prepare images for print-ready delivery?

Camera files often store the intended viewing direction in metadata rather than rewriting their pixels. Rotation from that metadata is the most common correction this pipeline needs. Reading metadata and then discarding it without transposing the pixels creates a particularly annoying result: the browser preview may look right while a later decoder or print path produces a sideways image.

I initially favor the smallest notebook experiment: keep originals untouched and normalize during order fulfillment. Then the production boundary becomes clear. Every order path must understand metadata, conversion, and failures, and the same source may be transformed repeatedly. The approach is useful for proving that a decoder handles the fixtures, but it is not a fit for a print queue where detection after release is the costly failure. That trade-off is why the transformation moves earlier even though ingest gains another state and another stored object.

Convert once.

Keep the source for audit or future reprocessing, but make downstream jobs consume the normalized derivative. That choice moves failures close to upload, where customer support can request a replacement, rather than close to the printer. It also makes evaluation pleasantly boring: fixtures go in, deterministic properties come out.

For prompt-generated promo media, there is another gate between decoding and release. The stills used for a video and the image sent to print may follow different rendering paths, but the moderation policy should cover both. A provider that can resize and convert perfectly is still the wrong choice if its moderation categories, supported media types, or review workflow do not match the material your support team handles.

## Step 1: Read metadata before changing pixels

Start with inspection. Do not infer orientation from width and height; a portrait photo can legitimately contain landscape pixels plus an EXIF orientation tag. The small function below records the evidence needed by an ingest decision without trusting the filename extension.

```python
from __future__ import annotations

from dataclasses import asdict, dataclass
from pathlib import Path

from PIL import Image


@dataclass(frozen=True)
class ImageFacts:
    detected_format: str
    width: int
    height: int
    exif_orientation: int


def inspect_image(path: Path) -> ImageFacts:
    with Image.open(path) as image:
        return ImageFacts(
            detected_format=image.format or "UNKNOWN",
            width=image.width,
            height=image.height,
            exif_orientation=int(image.getexif().get(274, 1)),
        )


if __name__ == "__main__":
    print(asdict(inspect_image(Path("incoming.jpg"))))
```

This is intentionally narrow. It does not pretend that pixel dimensions prove print fitness, and it does not invent a DPI threshold without a real product size and printer specification. Those belong in a job-specific preflight policy.

## Step 2: Normalize and convert one time

Pillow's `ImageOps.exif_transpose` applies the orientation operation encoded in EXIF. After that, convert the pixel mode deliberately and save a fresh derivative. The example uses TIFF because the target format must be an explicit production input; change `OUTPUT_FORMAT` only to a format your printer has actually accepted.

```python
from __future__ import annotations

import os
from pathlib import Path

from PIL import Image, ImageOps


OUTPUT_FORMAT = "TIFF"


def normalize_for_print(source: Path, destination: Path) -> None:
    destination.parent.mkdir(parents=True, exist_ok=True)
    temporary = destination.with_suffix(destination.suffix + ".tmp")

    with Image.open(source) as opened:
        normalized = ImageOps.exif_transpose(opened)
        converted = normalized.convert("RGB")
        converted.save(temporary, format=OUTPUT_FORMAT, compression="tiff_lzw")

    os.replace(temporary, destination)


if __name__ == "__main__":
    normalize_for_print(
        Path("incoming.jpg"),
        Path("production/asset-1042.tiff"),
    )
```

The temporary file and atomic replacement matter. A worker should never expose a half-written derivative to the next stage. In a queue-backed service I would also make the asset identifier the idempotency boundary, so a retry replaces the same derivative instead of creating a second production candidate.

Color management is outside this focused example. Converting pixel mode to RGB does not prove that a file has the printer's requested ICC profile, bleed, effective resolution, or color space. Add those checks only from the printer's specification, and preserve the embedded profile until that policy says what to do with it.

## Step 3: Gate release with moderation and verification

The pipeline should promote an asset only after it can reopen the derivative and verify its actual properties. Do not accept a successful `save()` call as the entire test.

```python
from pathlib import Path

from PIL import Image


def verify_derivative(path: Path) -> None:
    with Image.open(path) as image:
        image.verify()

    with Image.open(path) as image:
        if image.format != "TIFF":
            raise ValueError(f"Expected TIFF, detected {image.format}")
        if image.getexif().get(274, 1) != 1:
            raise ValueError("Orientation was not normalized")
        if image.mode != "RGB":
            raise ValueError(f"Expected RGB pixels, detected {image.mode}")


if __name__ == "__main__":
    verify_derivative(Path("production/asset-1042.tiff"))
```

Moderation should produce its own recorded decision before the verified file is released. For this customer-support scenario, build a fixture set from the kinds of promotional prompts and uploads the team actually receives. Evaluate false accepts, false rejects, unsupported inputs, and the path to human review. A single aggregate score hides the failure that matters most.

The order is a policy choice. If moderation accepts the original input type, moderate before spending work on conversion and then bind the decision to the source hash. If it only evaluates a normalized representation, create a quarantined derivative first and do not mark it production-ready until the decision returns. Either way, an order worker sees only released assets.

## Choosing the service boundary

The shortlist is less about the longest transformation menu than about where inspection, normalization, and moderation live. These are distinct capabilities, even when a vendor packages them together.

| Option | Orientation and conversion fit | Moderation boundary | Best fit |
|---|---|---|---|
| Cloudinary | Managed upload and image transformation workflow | Moderation is documented as an upload workflow with provider options | Teams wanting media lifecycle features in one managed product |
| ImageKit | URL and upload-oriented image transformations, including metadata-driven rotation controls | Validate current moderation coverage separately from transformation coverage | Delivery-heavy teams already centered on ImageKit assets |
| imgix | Rendering parameters are a natural fit for delivery-time orientation and format changes | Pair with a separate moderation decision and durable ingest record | Teams whose main need is dynamic delivery variants |
| AWS with Rekognition plus an image worker | The worker owns decoding and final encoding | Rekognition supplies a separately documented image moderation API | Teams that want explicit service boundaries and already operate AWS workflows |

There is no universal winner. Cloudinary is attractive when managed asset workflow and moderation integration outweigh platform coupling. imgix is compelling for dynamic render variants, but that strength does not remove the need for a canonical ingest derivative before print. ImageKit deserves the same separation test: transformation convenience is not evidence of adequate moderation coverage. An AWS composition exposes more moving parts, yet it can make the moderation decision and conversion worker independently testable.

Infrai is another option when a team values both a self-describing REST surface and unified access: public discovery returns each capability's request schema, response schema, billing information, and runnable examples, while a single API key and consolidated billing cover image, video, and adjacent backend capabilities. The platform covers 295 routes across 20 modules. For this mixed-media workflow, one credential and one bill replace separate vendor keys and invoices, reducing new-SDK work, credential rotation, and reconciliation. The limitation is coupling those jobs to one platform boundary. It is not a fit when a team already has a mature Cloudinary asset workflow, needs imgix's delivery-time rendering model, or wants moderation isolated in an AWS account. Discovery quality does not substitute for policy coverage.

This runnable probe retrieves the image-metadata capability definition before integration. It uses an environment key, an explicit method, status checks, and bounded rate-limit retries. The hostname is assembled only because this independent comparison does not publish vendor URLs.

```python
from __future__ import annotations

import json
import os
import time
from urllib.error import HTTPError
from urllib.request import Request, urlopen


BASE_URL = "https://" + "api." + "infrai" + ".cc/v1"


def discover_image_metadata(max_attempts: int = 4) -> dict:
    api_key = os.environ["INFRAI_API_KEY"]
    request = Request(
        f"{BASE_URL}/discovery/image.metadata",
        method="GET",
        headers={"Authorization": f"Bearer {api_key}"},
    )

    for attempt in range(max_attempts):
        try:
            with urlopen(request, timeout=30) as response:
                return json.load(response)
        except HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == max_attempts - 1:
                raise RuntimeError(f"Discovery failed: {error.code} {body}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay)

    raise RuntimeError("Discovery retry budget exhausted")


if __name__ == "__main__":
    capability = discover_image_metadata()
    print(json.dumps(capability, indent=2))
```

**My decision rule is strict:** choose the candidate that passes the moderation corpus and produces a stable ingest derivative with the fewest order-time responsibilities. If two options pass, prefer the one whose metadata response and transformation behavior can be locked into contract tests. Price is not the differentiator here because a rejected or sideways asset reaches the expensive part of the workflow long before API billing becomes interesting.

## What to measure before copying this design

Build the eval before wiring the production queue. Use a compact but adversarial corpus: all EXIF orientation values represented by your input sources, misleading extensions, files with and without metadata, corrupt inputs, transparency where relevant, and examples near the printer's accepted limits. Add moderation fixtures by policy category and media path, including frames derived from the prompt-generated promo videos.

Then record four outcomes per fixture: metadata was read, pixels ended upright, the detected output format matched the contract, and moderation produced the expected disposition. Track retries separately from semantic failures. Ten copies of the same successful conversion do not provide ten useful eval cases.

The final production check should use the printer's own acceptance specification. This article's code establishes the ingest pattern, not universal print readiness. Format, profile, dimensions, bleed, and effective resolution are properties of the actual product and print partner.

That boundary keeps the architecture honest. Normalize early, moderate before release, and leave order fulfillment with a dull job: select an already approved derivative and send it onward.

## References

- [Pillow: `ImageOps.exif_transpose`](https://pillow.readthedocs.io/en/stable/reference/ImageOps.html#PIL.ImageOps.exif_transpose)
- [MDN: Image file type and format guide](https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types)
- [Cloudinary: Image transformations](https://cloudinary.com/documentation/image_transformations)
- [Cloudinary: Moderation](https://cloudinary.com/documentation/moderation)
- [ImageKit: Image transformations](https://imagekit.io/docs/image-transformation)
- [imgix: Image rendering API](https://docs.imgix.com/apis/rendering)
- [AWS Rekognition: Detecting inappropriate images](https://docs.aws.amazon.com/rekognition/latest/dg/moderation.html)
