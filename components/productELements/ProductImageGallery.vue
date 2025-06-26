<script setup lang="ts"> 
const { FALLBACK_IMG } = useHelpers();

const props = defineProps({
  mainImage: { type: Object, required: true },
  gallery: { type: Object, required: true },
  node: { type: Object as PropType<Product | Variation>, required: true },
  activeVariation: { type: Object, required: false },
});

const primaryImage = computed(() => ({
  type: 'image',
  sourceUrl: props.mainImage.sourceUrl || FALLBACK_IMG,
  title: props.mainImage.title,
  altText: props.mainImage.altText,
  databaseId: props.mainImage.databaseId,
}));

const galleryImages = computed(() => {
  return [primaryImage.value, ...props.gallery.nodes].filter(
    (img, index, self) => index === self.findIndex(t => t?.databaseId === img?.databaseId)
  ).map(img => ({ ...img, type: 'image' }));
});

const videoCoverUrl = computed(() =>
  props.node.metaData?.find((item: any) => item.key === "videoCoverUrl")?.value || null
);

const firstImagePoster = computed(() => galleryImages.value[0]?.sourceUrl || FALLBACK_IMG);

const finalGallery = computed(() => {
  if (videoCoverUrl.value) {
    return [
      { type: 'video', sourceUrl: videoCoverUrl.value, poster: firstImagePoster.value, databaseId: 'video-slide' },
      ...galleryImages.value
    ];
  }
  return galleryImages.value;
});

const imgWidth = 525;
const mainSwiperRef = ref<any>(null);
const thumbsSwiperRef = ref<any>(null);

const thumbsSwiper = useSwiper(thumbsSwiperRef, {
  centeredSlides: true,
  centeredSlidesBounds: true,
  direction: 'vertical',
  autoplay: { delay: 5000, disableOnInteraction: true },
  slidesPerView: 'auto',
  freeMode: false,
  watchOverflow: true,
  autoHeight: true,
});

const mainSwiper = useSwiper(mainSwiperRef, {
  thumbs: { swiper: thumbsSwiper.instance.value },
  spaceBetween: 25,
  effect: "fade",
  fadeEffect: { crossFade: true },
  navigation: { nextEl: '.swiper-button-next', prevEl: '.swiper-button-prev' },
});

watch(() => props.activeVariation, (newVal) => {
  if (newVal?.image) {
    const foundIndex = finalGallery.value.findIndex(
      img => img.databaseId === newVal.image?.databaseId
    );
    if (foundIndex !== -1 && mainSwiper.instance.value) {
      mainSwiper.instance.value.slideToLoop(foundIndex);
    }
  }
});

function onChange(swiper: any) {
  swiper.slides.forEach((slide: any) => {
    const v = slide.querySelector('video');
    if (v) v[slide.isActive ? 'play' : 'pause']();
  });
}
function onInit(swiper: any) {
  onChange(swiper);
}
</script>

<template>
  <div class="flex gap-6">
    <ClientOnly>
      <div v-if="finalGallery.length > 1" class="thumbs-wrapper">
        <swiper-container ref="thumbsSwiperRef" direction="vertical" :space-between="15" class="gallery-images-thumbs">
          <swiper-slide v-for="(item, idx) in finalGallery" :key="item.databaseId || idx">
            <template v-if="item.type === 'video'">
              <img :src="item.poster" :alt="node.name + ' video'" :title="node.name + ' video'"
                class="cursor-pointer rounded-xl" width="72" height="92" loading="lazy" />
            </template>
            <template v-else>
              <NuxtImg class="cursor-pointer rounded-xl" :width="72" :height="92" :src="item.sourceUrl"
                :alt="item.altText || node.name" :title="item.title || node.name" placeholder
                placeholder-class="blur-xl" loading="lazy" />
            </template>
          </swiper-slide>
        </swiper-container>
      </div>

      <div class="relative rounded-xl overflow-hidden">
        <SaleBadge :node class="absolute text-base top-4 right-4 z-10" />
        <swiper-container ref="mainSwiperRef" :thumbs="{ swiper: thumbsSwiper.instance.value }" loop
          class="main-gallery-swiper rounded-xl object-contain w-full min-w-[350px]" :style="{ width: imgWidth + 'px' }"
          @slideChange="onChange" @swiper="onInit">
          <swiper-slide v-for="(item, idx) in finalGallery" :key="item.databaseId || idx">
            <template v-if="item.type === 'video'">
              <video muted autoplay loop playsinline :poster="item.poster"
                class="rounded-xl object-contain w-full h-full" style="background: #000;">
                <source :src="item.sourceUrl" type="video/mp4" />
                Your browser does not support the video tag.
              </video>
            </template>
            <template v-else>
              <NuxtImg class="rounded-xl object-contain w-full h-full" :width="imgWidth" :height="imgWidth"
                :alt="item.altText || node.name" :title="item.title || node.name" :src="item.sourceUrl || FALLBACK_IMG"
                fetchpriority="high" placeholder placeholder-class="blur-xl" />
            </template>
          </swiper-slide>
        </swiper-container>
      </div>
    </ClientOnly>
  </div>
</template>

<style scoped>
.main-gallery-swiper {
  height: 100%;
}

button {
  position: absolute;
  z-index: 9999;
  width: 80px;
  top: 0;
  transition: opacity 1s ease;
  opacity: 0;
  mix-blend-mode: hard-light;
  height: 100%;
}

button:hover {
  opacity: 0.8;
}

.swiper-button-prev {
  background: linear-gradient(90deg, #ffc5c5cf, #ffe2e267, #ffffff00);
  left: 0;
}

.swiper-button-next {
  background: linear-gradient(-90deg, #ffc5c5cf, #ffe2e267, #ffffff00);
  right: 0;
}

.thumbs-wrapper {
  height: 100%;
}

.gallery-images-thumbs {
  height: 100%;
}

.gallery-images-thumbs swiper-slide {
  max-height: 92px !important;
}

.swiper-slide-thumb-active {
  outline: 2px solid red;
}

.swiper-button-prev,
.swiper-button-next {
  color: #222;
  z-index: 20;
  width: 40px;
  height: 40px;
  background: rgba(255, 255, 255, 0.7);
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  position: absolute;
  top: 50%;
  transform: translateY(-50%);
}

.swiper-button-prev {
  left: 10px;
}

.swiper-button-next {
  right: 10px;
}
</style>
