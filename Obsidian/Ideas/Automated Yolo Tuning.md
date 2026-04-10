# Auto-Labeling YOLO Training Data with Foundation Models

## The Problem

Fine-tuning YOLO models requires labeled training data — hundreds or thousands of images with bounding boxes drawn around target objects. Manual labeling is tedious, expensive, and doesn't scale. A single person can label roughly 50-100 images per hour depending on complexity, meaning a 1,000-image dataset could take 10-20 hours of focused work.

Foundation models (large vision-language models and zero-shot detectors) can automate the bulk of this work, shifting the human role from **labeling** to **reviewing**, which is dramatically faster.

---

## Foundation Models That Can Help

### Zero-Shot Object Detectors

These models detect objects based on text prompts without any training on your specific data.

**Grounding DINO**
- Open-source zero-shot object detector by IDEA Research
- Input: an image + a text prompt (e.g., "metal part", "plier head", "screwdriver blade")
- Output: bounding boxes with confidence scores
- No training required — it generalizes from its pretraining
- Best option for generating YOLO-format bounding boxes directly
- GitHub: https://github.com/IDEA-Research/GroundingDINO

**Florence-2 (Microsoft)**
- Vision-language model that outputs bounding boxes, segmentation masks, and captions
- Can be fine-tuned with very few examples (few-shot)
- Lighter weight than Grounding DINO, runs well on consumer GPUs
- Good alternative if Grounding DINO struggles with your specific objects

### Segmentation Models

These refine bounding boxes into precise object outlines (useful if you later need segmentation data).

**SAM 2 (Meta — Segment Anything Model 2)**
- Given a bounding box or point prompt, produces pixel-perfect segmentation masks
- Works on both images and video
- Pair with Grounding DINO: DINO finds the box → SAM refines the boundary
- GitHub: https://github.com/facebookresearch/segment-anything-2

### Vision LLMs (Claude, GPT-4o, Gemini)

These can classify images and describe object locations but don't natively output precise bounding box coordinates in YOLO format. Their role in the pipeline is:

- **Classification**: "Is there a part in this image? What type?"
- **Quality review**: "Does this bounding box correctly surround the target object?"
- **Edge case triage**: Send ambiguous images to a vision LLM for judgment calls
- **Video frame selection**: Analyze video frames to identify which ones contain useful training examples

They work best as a **classification and review layer** rather than the primary box-drawing tool.

---

## Recommended Auto-Labeling Pipeline

### Step 1: Collect Raw Images

Capture images that represent the full variety of conditions the model will encounter in production.

**From video:**
```bash
# Extract 1 frame per second from a 30-minute video = 1,800 frames
ffmpeg -i station_recording.mp4 -vf fps=1 frames/frame_%04d.jpg
```

**What to capture:**
- All part types the model needs to detect
- Different operators (body positions, hand placements vary)
- All lighting conditions (morning shift, afternoon, night shift, overhead vs. natural light)
- Parts at different stages of the cycle (arriving, in fixture, being worked on, leaving)
- Edge cases: empty fixture, partial occlusion, operator reaching across frame

**How many images:**
- Target 200-500 raw frames per station type
- After deduplication and filtering, you'll label ~100-300

**Deduplication:** Many adjacent frames from video will be nearly identical. Filter to keep only frames with meaningful visual differences:
```bash
# Simple approach: keep every Nth frame
ffmpeg -i station_recording.mp4 -vf "fps=0.2" frames/frame_%04d.jpg  # 1 frame every 5 seconds
```

### Step 2: Auto-Label with Grounding DINO

Run Grounding DINO on your image set with text prompts describing your target objects.

**Prompt engineering matters:**
- Be specific: "metal multi-tool on workbench" works better than "tool"
- Try multiple prompts and take the union of results
- Use prompts that describe visual appearance, not function: "silver rectangular metal object" may work better than "Leatherman Wave"

**Example prompts by use case:**
- Assembly station: "metal part", "workpiece in fixture", "assembled product"
- Packaging station: "cardboard box", "product in packaging", "sealed package"
- Quality inspection: "part under magnifier", "part on inspection fixture"

**Confidence threshold:** Start with 0.3 (permissive) and filter later during review. It's easier to delete false positives than to find missed detections.

**Output:** Bounding boxes in YOLO format per image:
```
# class_id center_x center_y width height (all normalized 0-1)
0 0.512 0.434 0.231 0.187
```

### Step 3: Human Review Pass

This is where the time savings happen. Instead of drawing boxes from scratch, you're reviewing and correcting pre-generated labels.

**Tools for review:**
- **Roboflow** — Upload images + auto-labels, use their annotation editor to fix errors, export directly to YOLO format. Free tier handles this scale.
- **Label Studio** — Self-hosted open-source alternative. Import pre-annotations, review and correct.
- **CVAT** — Another open-source option with good pre-annotation support.

**What to look for during review:**
- False positives: boxes around things that aren't the target object (delete them)
- Missed detections: objects with no box (add them manually)
- Loose boxes: box is around the right thing but too big/small (adjust it)
- Wrong class: box is correct but labeled as the wrong class (re-label)

**Expected effort:** Reviewing and correcting auto-labels takes roughly 3-5x less time than labeling from scratch. A 300-image dataset that would take 4-6 hours to label manually can be reviewed in 1-2 hours.

### Step 4: Augment and Split

Before training, augment and split the dataset.

**Augmentations (applied automatically during training or via Roboflow):**
- Brightness/contrast variation (simulates lighting changes)
- Small rotations (±5-10°) to handle slight camera angle differences
- Horizontal flip (if applicable to your objects)
- Mosaic augmentation (built into YOLO training — combines 4 images into one)
- Noise injection (simulates camera quality variation)

**Dataset split:**
- Train: 70%
- Validation: 20%
- Test: 10%

### Step 5: Train YOLO

```bash
# Using ultralytics library
yolo detect train data=dataset.yaml model=yolov8s.pt epochs=100 imgsz=640
```

**Model size selection:**
- YOLOv8n (nano): fastest, least accurate — good for edge devices with limited compute
- YOLOv8s (small): best balance of speed and accuracy — recommended starting point
- YOLOv8m (medium): more accurate, slower — use if small isn't hitting accuracy targets
- YOLOv8l/x (large/xlarge): highest accuracy, requires significant GPU — rarely needed for fixed-camera industrial use

**Training tips:**
- Start from pretrained COCO weights (the default) — transfer learning dramatically reduces data requirements
- 50-100 epochs is usually sufficient for fine-tuning with <1000 images
- Watch validation mAP — if it plateaus, stop training
- If mAP is below 80%, you likely need more/better training data, not more epochs

---

## Alternative Approach: Autodistill

Roboflow's **Autodistill** library wraps this entire pipeline into a streamlined workflow. It uses a "teacher-student" paradigm: a large foundation model (teacher) labels data, then a small efficient model (student, i.e., YOLO) is trained on those labels.

```python
# Pseudocode — Autodistill pipeline
from autodistill_grounded_sam import GroundedSAM
from autodistill.detection import CaptionOntology

# Define what to detect via text prompts
ontology = CaptionOntology({
    "metal part on fixture": "part",
    "empty fixture": "empty_fixture",
    "operator hands in work zone": "hands"
})

# Teacher model auto-labels your images
teacher = GroundedSAM(ontology=ontology)
teacher.label(input_folder="./raw_images", output_folder="./labeled_dataset")

# Student model (YOLO) trains on the auto-labels
from autodistill_yolov8 import YOLOv8
student = YOLOv8("yolov8s.pt")
student.train("./labeled_dataset/data.yaml", epochs=100)
```

This collapses Steps 2-5 into a few lines of code. The tradeoff is less control over the review step — you're trusting the foundation model's labels more directly. For a first pass or POC, this is extremely fast. For production models, adding a human review step (Step 3) after Autodistill's labeling will improve quality.

---

## When to Use Which Approach

| Scenario | Recommended Approach |
|----------|---------------------|
| Quick POC, just need something working | Autodistill end-to-end, skip manual review |
| Production model, accuracy matters | Grounding DINO → Roboflow review → YOLO training |
| Objects look very different from COCO classes | Florence-2 (few-shot) or more manual labeling |
| Need segmentation masks, not just boxes | Grounding DINO + SAM 2 |
| Video data, need to track objects across frames | SAM 2 (native video support) |
| Classification only (no bounding boxes needed) | Vision LLM (Claude/GPT-4o) on sampled frames |

---

## Expected Effort and Timeline

| Task | Manual Approach | Auto-Label Approach |
|------|----------------|-------------------|
| Collect images (30 min video, extract frames) | 1-2 hours | 1-2 hours |
| Label 300 images | 4-6 hours | 30-60 min (auto) + 1-2 hours (review) |
| Train YOLO model | 1-2 hours | 1-2 hours |
| Validate and iterate | 2-4 hours | 2-4 hours |
| **Total per station type** | **8-14 hours** | **4-8 hours** |

The time savings compound as you add more station types — the auto-labeling pipeline is reusable with just new text prompts.

---

## Key Takeaways

1. **Grounding DINO is the workhorse** — it generates bounding boxes from text prompts with no training, directly in YOLO-compatible format.
2. **Vision LLMs (Claude, GPT-4o) are best for classification and review**, not for generating bounding box coordinates.
3. **Human review is still important** — auto-labels get you 70-85% of the way there. A quick review pass catches the rest and significantly improves final model quality.
4. **You need far less data than you think** — 100-300 well-labeled images per station type, with augmentation, can produce a model with 90%+ mAP for fixed-camera industrial detection.
5. **Autodistill is the fastest path to a working model** — use it for POC, add human review for production.
6. **Start simple** — pretrained YOLO on COCO already detects "person" at high accuracy. You may only need custom training for detecting specific parts, not for detecting operators.
