---
name: wagtail-accessibility
description: 'Build accessible Wagtail sites. Use this any time the user mentions Wagtail accessibility, a11y, WCAG compliance, accessible Wagtail admin, ARIA in Wagtail templates, accessible images in Wagtail, keyboard navigation, screen reader support, or accessibility testing for Wagtail projects.'
---

# Wagtail Accessibility (a11y)

## Orientation

Accessibility in Wagtail spans two distinct areas:
1. **The Wagtail admin** — how content editors with disabilities use the CMS
2. **The public-facing site** — how visitors consume the content your Wagtail site serves

Wagtail's admin has a strong baseline (WCAG 2.1 AA target, keyboard-navigable, screen-reader-friendly), but the public site's accessibility is entirely up to your templates — Wagtail gives you the content tools; you build the accessible output.

## Public Site Accessibility

### Images — always require `alt` text

Make `alt` text mandatory on image fields:

```python
from wagtail.images.models import AbstractImage

class CustomImage(AbstractImage):
    @property
    def alt_text(self):
        return self.default_alt_text or self.title

# Template
{% image page.hero_image fill-1200x600 alt=page.hero_image.alt_text %}
```

Never render an image without `alt`. For decorative images, use `alt=""`. For content images, the alt text should describe the image's purpose, not its literal content.

```python
# In the model, encourage editors:
alt_text = models.CharField(
    max_length=255,
    blank=True,
    help_text="Describe the image for screen readers. Leave blank for decorative images."
)
```

### Semantic HTML in Rich Text

Wagtail's rich text editor (Draftail) can be configured to enforce heading hierarchy:

```python
# wagtail_hooks.py
from wagtail import hooks

@hooks.register('register_rich_text_features')
def register_accessible_rich_text(features):
    # Remove h1 from rich text (h1 should be the page title, not inline)
    features.default_features.remove('h1')
    # Force editors to use proper heading levels
```

### ARIA in templates

```django
<!-- Navigation with ARIA -->
<nav aria-label="Main navigation">
    <ul role="list">
        {% for item in menu_items %}
            <li>
                <a href="{{ item.url }}" {% if item.active %}aria-current="page"{% endif %}>
                    {{ item.title }}
                </a>
            </li>
        {% endfor %}
    </ul>
</nav>

<!-- Skip to content -->
<a href="#main-content" class="skip-link">Skip to main content</a>

<main id="main-content">
    {{ page.body }}
</main>

<!-- Forms with proper labels -->
<form>
    <label for="search-input">Search</label>
    <input id="search-input" type="search" name="q" aria-label="Search the site">
    <button type="submit" aria-label="Submit search">Search</button>
</form>
```

### Accessible StreamField blocks

When building custom StreamField blocks, ensure the template output is accessible:

```python
class CardBlock(blocks.StructBlock):
    image = ImageChooserBlock()
    title = blocks.CharBlock()
    link = blocks.PageChooserBlock()
    description = blocks.TextBlock()

    class Meta:
        template = 'blocks/card.html'
```

```html
<!-- blocks/card.html -->
<article class="card">
    {% if value.link %}
        <a href="{{ value.link.url }}">
            {% image value.image fill-400x300 alt=value.image.alt_text %}
            <h3>{{ value.title }}</h3>
        </a>
    {% endif %}
    <p>{{ value.description }}</p>
</article>
```

### Forms accessibility

```django
{% for field in form %}
    <div class="field">
        <label for="{{ field.id_for_label }}">
            {{ field.label }}
            {% if field.field.required %}
                <span aria-hidden="true">*</span>
                <span class="sr-only">(required)</span>
            {% endif %}
        </label>
        {{ field }}
        {% if field.errors %}
            <div class="error" id="{{ field.id_for_label }}-error" role="alert">
                {{ field.errors }}
            </div>
        {% endif %}
        {% if field.help_text %}
            <div class="help-text" id="{{ field.id_for_label }}-help">
                {{ field.help_text }}
            </div>
        {% endif %}
    </div>
{% endfor %}
```

### Color contrast

Set defaults in your design tokens that meet WCAG AA (4.5:1 for normal text, 3:1 for large text). Wagtail doesn't enforce color contrast — it's a template responsibility.

For editor-chosen colors (StreamField color picker), validate in the model:

```python
from django.core.exceptions import ValidationError

class ColorBlock(blocks.StructBlock):
    color = blocks.CharBlock()

    def clean(self, value):
        result = super().clean(value)
        if not meets_contrast_ratio(result['color'], '#ffffff', 4.5):
            raise ValidationError('Text color must have at least 4.5:1 contrast on white')
        return result
```

## Wagtail Admin Accessibility

### Built-in features

Wagtail's admin already provides:
- **Keyboard navigation** — all admin interfaces are keyboard-operable
- **Focus management** — modal dialogs trap focus, StreamField blocks have visible focus indicators
- **Screen reader support** — ARIA labels on edit panels, status messages, and modals
- **Color contrast** — admin UI meets WCAG 2.1 AA

### Custom admin accessibility

When building custom admin views or panels, follow Wagtail's patterns:

```python
# Custom admin view
from wagtail.admin.views.generic import IndexView

class CustomIndexView(IndexView):
    page_title = "Products"
    header_icon = "snippet"
    
    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        context['aria_label'] = 'Product listing'
        return context
```

### Admin accessibility testing

Wagtail's admin is tested with automated a11y checks in CI. For custom admin work, add similar tests:

```python
# tests/test_admin_a11y.py
from django.test import TestCase

class AdminAccessibilityTest(TestCase):
    def setUp(self):
        from django.contrib.auth.models import User
        self.user = User.objects.create_superuser('admin', 'a@b.com', 'pass')
        self.client.login(username='admin', password='pass')
    
    def test_page_editor_has_skiplink(self):
        response = self.client.get(f'/admin/pages/{self.page.id}/edit/')
        self.assertContains(response, 'skip-link')
    
    def test_admin_pages_have_lang_attribute(self):
        response = self.client.get('/admin/')
        self.assertContains(response, 'lang="en"')
```

## Testing Public Site Accessibility

### Automated tools

```bash
# Use axe-core CLI for automated checks
npx @axe-core/cli https://staging.example.com --tags wcag2a,wcag2aa

# pa11y for CI integration
npx pa11y https://staging.example.com
```

### Manual testing checklist

- [ ] Tab through every interactive element — can you reach and operate everything with keyboard only?
- [ ] Navigate with a screen reader (VoiceOver, NVDA) — can you understand all content and complete core tasks?
- [ ] Zoom to 200% — does layout still work without horizontal scrolling?
- [ ] Disable images — is all content still understandable?
- [ ] Test with a color blindness simulator

## Resources

- **WCAG 2.1 AA Quick Reference**: https://www.w3.org/WAI/WCAG21/quickref/
- **axe DevTools**: Browser extension for in-context a11y testing
- **Wagtail accessibility docs**: https://docs.wagtail.org/en/stable/contributing/accessibility.html
