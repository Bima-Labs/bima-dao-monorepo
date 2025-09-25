<script setup lang="ts">
import { computed } from 'vue'; // Import 'computed'
import { autoLinkText } from '@/helpers/utils';
import { Vote } from '@/types';

const props = defineProps<{
  open: boolean;
  vote: Vote | null;
}>();

defineEmits<{
  (e: 'close'): void;
}>();

/**
 * A basic HTML sanitizer that aims to be safe for content primarily containing text
 * and only allows a very limited set of HTML, specifically <a> tags created by autoLinkText.
 * It removes script tags and strips attributes from all other tags.
 * For robust security in production, a dedicated library like DOMPurify is strongly recommended.
 */
function safeHtmlForReason(html: string): string {
  if (!html) return '';

  // Use a DOMParser for more reliable parsing and sanitization than regex alone.
  const doc = new DOMParser().parseFromString(html, 'text/html');
  const body = doc.body;

  // Remove all script tags
  body.querySelectorAll('script').forEach(s => s.remove());

  // Iterate through all elements to sanitize attributes and potentially remove unsafe tags
  // Query all elements for processing
  const allElements = Array.from(body.querySelectorAll('*')); // Convert NodeList to Array for safe iteration
  allElements.forEach(el => {
    if (el.tagName.toLowerCase() === 'a') {
      // For <a> tags, only allow 'href' attribute and ensure its value is safe
      const href = el.getAttribute('href');
      const safeHref = (href && (href.startsWith('http://') || href.startsWith('https://') || href.startsWith('mailto:')))
        ? href
        : '#'; // Default to '#' or strip if unsafe

      // Create a new <a> element to strip all other attributes
      const newA = doc.createElement('a');
      if (safeHref !== '#') { // Only add href if it's considered safe
        newA.setAttribute('href', safeHref);
      }
      newA.textContent = el.textContent;
      el.replaceWith(newA);
    } else {
      // For all other tags, replace them with their text content to strip all tags and attributes.
      // This ensures only plain text (and safe <a> tags) remain.
      el.replaceWith(doc.createTextNode(el.textContent || ''));
    }
  });

  return body.innerHTML;
}


const sanitizedReasonHtml = computed(() => {
  const reason = props.vote?.reason || '';
  const linkedText = autoLinkText(reason);
  // Apply a basic sanitization to ensure only safe HTML (primarily <a> tags from autoLinkText) remains.
  // IMPORTANT: For production, a robust HTML sanitization library (e.g., DOMPurify) is strongly recommended.
  return safeHtmlForReason(linkedText);
});
</script>

<template>
  <UiModal :open="open" @close="$emit('close')">
    <template #header>
      <h3 v-text="'Reason'" />
    </template>
    <div
      class="vote-reason p-4 whitespace-pre-line text-skin-link break-words"
      v-html="sanitizedReasonHtml"
    />
  </UiModal>
</template>

<style class="scss" scoped>
.vote-reason:deep() a {
  text-decoration: underline;
}
</style>