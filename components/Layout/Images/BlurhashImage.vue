<template>
  <div v-if="isValidTag()" ref="img" class="absolute">
    <transition-group name="fade" mode="in-out">
      <blurhash-canvas
        v-if="canBeBlurhashed()"
        key="canvas"
        :hash="getHash()"
        :width="width"
        :height="height"
        :punch="punch"
        class="absolute"
      />
      <img
        v-if="!lazy"
        v-show="!loading"
        key="image"
        class="absolute"
        :src="image"
        v-bind="$attrs"
        @error="onError"
        @load="onLoad"
      />
      <lazy-image
        v-else
        key="image"
        class="absolute"
        :src="image"
        v-bind="$attrs"
        @error="onError"
      />
    </transition-group>
  </div>
</template>

<script lang="ts">
import Vue from 'vue';
import {
  BaseItemDto,
  BaseItemDtoImageBlurHashes,
  BaseItemPerson,
  ImageType
} from '@jellyfin/client-axios';
import imageHelper from '~/mixins/imageHelper';

const excludedTypes = [ImageType.Logo];

export default Vue.extend({
  mixins: [imageHelper],
  props: {
    item: {
      type: Object as () => BaseItemDto | BaseItemPerson,
      required: true
    },
    width: {
      type: Number,
      default: 32
    },
    height: {
      type: Number,
      default: 32
    },
    punch: {
      type: Number,
      default: 1
    },
    type: {
      type: String as () => ImageType,
      default: ImageType.Primary
    },
    lazy: {
      type: Boolean,
      default: () => {
        return true;
      }
    }
  },
  data() {
    return {
      image: '',
      loading: true
    };
  },
  mounted(): void {
    const img = this.$refs.img as HTMLElement;
    // We don't pass the item itself as we already did all the tags checking in this component,
    // so doing it again in the mixin is useless.
    this.image = this.getImageUrlForElement(this.type, {
      itemId: this.item.Id as string,
      element: img
    });
  },
  methods: {
    isPerson(obj: BaseItemDto | BaseItemPerson): obj is BaseItemPerson {
      if ((obj as BaseItemPerson).Role) {
        return true;
      }
      return false;
    },
    isValidTag(): boolean {
      if (!this.isPerson(this.item)) {
        if (
          this.item?.ImageTags?.[this.type] ||
          (this.type === ImageType.Backdrop && this.item?.BackdropImageTags)
        ) {
          return true;
        } else {
          this.onError();
        }
      } else if (
        this.item?.PrimaryImageTag &&
        this.type === ImageType.Primary
      ) {
        return true;
      } else {
        this.onError();
      }
      return false;
    },
    canBeBlurhashed(): boolean {
      if (excludedTypes.includes(this.type) || !this.item?.ImageBlurHashes) {
        return false;
      } else if (!this.isPerson(this.item)) {
        if (
          this.type === ImageType.Backdrop &&
          !this.item?.BackdropImageTags &&
          this.item.ImageBlurHashes?.Backdrop &&
          this.item.ImageBlurHashes?.Backdrop?.[
            (this.item.BackdropImageTags as Array<string>)[0]
          ]
        ) {
          return true;
        } else if (
          this.item?.ImageBlurHashes?.[this.type] &&
          this.item?.ImageTags?.[this.type] &&
          this.item?.ImageBlurHashes?.[this.type]?.[
            this.item?.ImageTags?.[this.type]
          ]
        ) {
          return true;
        }
      } else if (
        this.item?.PrimaryImageTag &&
        this.item?.ImageBlurHashes?.Primary &&
        this.item?.ImageBlurHashes?.Primary?.[this.item?.PrimaryImageTag]
      ) {
        return true;
      }
      return false;
    },
    getHash(): string {
      if (!this.isPerson(this.item)) {
        if (this.type === ImageType.Backdrop) {
          const tag = (this.item?.BackdropImageTags as Array<string>)[0];

          return ((this.item?.ImageBlurHashes as BaseItemDtoImageBlurHashes)
            .Backdrop as Record<string, string>)[tag];
        } else {
          return ((this.item?.ImageBlurHashes as BaseItemDtoImageBlurHashes)[
            this.type
          ] as Record<string, string>)[
            (this.item?.ImageTags as Record<string, string>)[this.type]
          ];
        }
      } else {
        return ((this.item?.ImageBlurHashes as BaseItemDtoImageBlurHashes)[
          this.type
        ] as Record<string, string>)[this.item?.PrimaryImageTag as string];
      }
    },
    onError(): void {
      this.$emit('error');
    },
    onLoad(): void {
      this.loading = false;
    }
  }
});
</script>

<style lang="scss" scoped>
.fade-enter-active,
.fade-leave-active {
  transition: opacity 0.15s;
}

.fade-enter,
.fade-leave-to {
  opacity: 0;
}

.absolute {
  height: 100%;
  width: 100%;
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
}

img {
  object-fit: cover;
}
</style>
