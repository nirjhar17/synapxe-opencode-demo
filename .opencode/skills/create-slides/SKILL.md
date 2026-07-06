---
name: create-slides
description: >-
  Create a PowerPoint (.pptx) presentation from notes, topics, or any input.
  Use when the user asks to create slides, make a deck, build a presentation,
  generate a PowerPoint, or turn notes into slides.
---

# Create Slides

Generate a professional PowerPoint (.pptx) presentation using python-pptx. The presentation is saved locally as a file the user can download.

## Example Trigger Prompts

- "Create slides from these meeting notes"
- "Make a presentation about our migration plan"
- "Turn these notes into a deck"
- "Build a PowerPoint for the Q3 review"

## Instructions

Follow these steps in order:

1. Read and understand the input (notes, topic, or content provided by the user).
2. Plan the slide structure. Aim for 6-10 slides. Every deck must include:
   - Title slide (topic + subtitle/date)
   - Agenda slide (3-5 items)
   - Content slides (one key point per slide)
   - Summary/Next Steps slide
3. Write a Python script using python-pptx to generate the .pptx file.
4. Execute the script to create the file.
5. Tell the user the file path so they can download it.

## Critical: Python Version

ALWAYS use `python3.11` to run the script, NOT `python3`. The system python3 is too old. Example:

```bash
python3.11 build_slides.py
```

## Slide Writing Rules

- Headlines should be assertions, not labels. "Migration Cuts Deployment Time by 60%" is better than "Migration Results".
- Keep bullet points to 4-5 per slide maximum.
- Each bullet should be one line, under 15 words.
- Use short, active sentences. No jargon.
- If content is too long for one slide, split into two slides.

## Python Script Template

Use this pattern for the script. Save it as `build_slides.py` and run it.

```python
from pptx import Presentation
from pptx.util import Inches, Pt, Emu
from pptx.dml.color import RGBColor
from pptx.enum.text import PP_ALIGN, MSO_ANCHOR

prs = Presentation()
prs.slide_width = Inches(13.333)
prs.slide_height = Inches(7.5)

# Color palette
BG_DARK = RGBColor(0x15, 0x15, 0x15)
BG_CARD = RGBColor(0x1E, 0x29, 0x3B)
WHITE = RGBColor(0xFF, 0xFF, 0xFF)
LIGHT_GRAY = RGBColor(0xA0, 0xA0, 0xA0)
ACCENT_RED = RGBColor(0xEE, 0x00, 0x00)
ACCENT_BLUE = RGBColor(0x3B, 0x82, 0xF6)
ACCENT_GREEN = RGBColor(0x22, 0xC5, 0x5E)
ACCENT_TEAL = RGBColor(0x37, 0xA3, 0xA3)


def set_slide_bg(slide, color):
    background = slide.background
    fill = background.fill
    fill.solid()
    fill.fore_color.rgb = color


def add_textbox(slide, left, top, width, height, text,
                font_size=18, color=WHITE, bold=False, alignment=PP_ALIGN.LEFT,
                font_name="Calibri"):
    txBox = slide.shapes.add_textbox(left, top, width, height)
    tf = txBox.text_frame
    tf.word_wrap = True
    p = tf.paragraphs[0]
    p.text = text
    p.font.size = Pt(font_size)
    p.font.color.rgb = color
    p.font.bold = bold
    p.font.name = font_name
    p.alignment = alignment
    return tf


def add_bullet_slide(prs, title, bullets, accent_color=ACCENT_BLUE):
    slide = prs.slides.add_slide(prs.slide_layouts[6])  # blank
    set_slide_bg(slide, BG_DARK)

    # Accent bar at top
    shape = slide.shapes.add_shape(1, 0, 0, prs.slide_width, Inches(0.08))
    shape.fill.solid()
    shape.fill.fore_color.rgb = accent_color
    shape.line.fill.background()

    # Title
    add_textbox(slide, Inches(0.8), Inches(0.4), Inches(11), Inches(0.8),
                title, font_size=32, bold=True, color=WHITE)

    # Bullets
    txBox = slide.shapes.add_textbox(Inches(0.8), Inches(1.5),
                                      Inches(11), Inches(5.0))
    tf = txBox.text_frame
    tf.word_wrap = True
    for i, bullet in enumerate(bullets):
        p = tf.paragraphs[0] if i == 0 else tf.add_paragraph()
        p.text = bullet
        p.font.size = Pt(20)
        p.font.color.rgb = RGBColor(0xDD, 0xDD, 0xDD)
        p.font.name = "Calibri"
        p.space_after = Pt(12)
    return slide


# --- Build your slides here ---

# Slide 1: Title
slide = prs.slides.add_slide(prs.slide_layouts[6])
set_slide_bg(slide, ACCENT_RED)
add_textbox(slide, Inches(0.8), Inches(2.0), Inches(11), Inches(1.5),
            "Your Title Here", font_size=44, bold=True, color=WHITE)
add_textbox(slide, Inches(0.8), Inches(3.8), Inches(11), Inches(0.8),
            "Subtitle or Date", font_size=22, color=RGBColor(0xFF, 0xCC, 0xCC))

# Slide 2: Agenda
add_bullet_slide(prs, "Agenda", [
    "First topic",
    "Second topic",
    "Third topic",
])

# Slide 3+: Content slides
add_bullet_slide(prs, "Key Finding", [
    "Point one with supporting detail",
    "Point two with supporting detail",
    "Point three with supporting detail",
])

# Last slide: Next Steps
slide = prs.slides.add_slide(prs.slide_layouts[6])
set_slide_bg(slide, BG_DARK)
shape = slide.shapes.add_shape(1, 0, 0, prs.slide_width, Inches(0.08))
shape.fill.solid()
shape.fill.fore_color.rgb = ACCENT_GREEN
shape.line.fill.background()
add_textbox(slide, Inches(0.8), Inches(2.5), Inches(11), Inches(1.5),
            "Next Steps", font_size=40, bold=True, color=WHITE,
            alignment=PP_ALIGN.CENTER)

# Save
prs.save("presentation.pptx")
print("Saved: presentation.pptx")
```

## Rules

- Always use blank slide layout (index 6) and build from scratch.
- Always set a dark background on every slide.
- Always add a colored accent bar at the top of content slides.
- Title slide should use a bold accent color (red) as full background.
- Save the file in the current project directory.
- File name should be descriptive: "migration-strategy.pptx" not "slides.pptx".
- After saving, tell the user the exact file path and how to download it from Dev Spaces.
- If python-pptx is not installed, run `python3.11 -m pip install python-pptx` first.
- ALWAYS use `python3.11` to run the script. NEVER use `python3` or `python`.
