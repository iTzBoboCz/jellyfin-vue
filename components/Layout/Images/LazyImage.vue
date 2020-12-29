<template>
  <img :data-src="src" :alt="alt" class="lazy-hidden" @error="onError" />
</template>

<script lang="ts">
import Vue from 'vue';
import { sanitize } from 'dompurify';

/**
 * Swaps the 'data-src' attribute with 'src' attribute.
 *
 * @param {HTMLElement} elem - DOM node of the target img element
 * @param {string} url - url of the image
 *
 */
function fillImageElement(elem: HTMLElement, url: string): void {
  const preloaderImg = new Image();
  preloaderImg.src = url;

  preloaderImg.addEventListener('load', () => {
    requestAnimationFrame(() => {
      elem.setAttribute('src', url);
      elem.removeAttribute('data-src');
      elem.classList.remove('lazy-hidden');
      elem.classList.add('lazy-show');
    });
  });
}

/**
 * Swaps the 'src' attribute with 'data-src' attribute.
 *
 * @param {HTMLElement} elem - DOM node of the target img element
 *
 */
function emptyImageElement(elem: HTMLElement): void {
  const url = sanitize(elem.getAttribute('src') as string);

  elem.removeAttribute('src');
  elem.setAttribute('data-src', url);
  elem.classList.remove('lazy-show');
  elem.classList.add('lazy-hidden');
}

/**
 * Global callback function for all the observed lazy loaded images
 *
 * @param {IntersectionObserverEntry} entry - entry of the intersected element.
 */
function onIntersect(entry: IntersectionObserverEntry): void {
  const target = entry.target as HTMLElement;
  const isIntersecting = entry.isIntersecting;

  if (target) {
    const source = target.getAttribute('data-src');
    if (!isIntersecting && !source) {
      requestAnimationFrame(() => {
        emptyImageElement(target);
      });
    } else if (source && isIntersecting) {
      fillImageElement(target, sanitize(source));
    }
  }
}

const observer = new IntersectionObserver((entries) => {
  entries.forEach(
    (entry) => {
      onIntersect(entry);
    },
    { rootMargin: '25%' }
  );
});

const observedTargets: Array<Element> = [];

export default Vue.extend({
  props: {
    src: {
      type: String,
      required: true
    },
    alt: {
      type: String,
      required: false,
      default: ''
    }
  },
  mounted() {
    observer.observe(this.$el);
    observedTargets.push(this.$el);
  },
  destroyed() {
    observer.unobserve(this.$el);
    observedTargets.splice(observedTargets.indexOf(this.$el), 1);
  },
  methods: {
    onError(): void {
      this.$emit('error');
    }
  }
});
</script>

<style lang="scss" scoped>
.lazy-show {
  opacity: 1;
  transition: opacity 0.15s;
}

.lazy-hidden {
  opacity: 0;
}
</style>
