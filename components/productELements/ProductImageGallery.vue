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
      slidesPerView: 7,
      freeMode: false,
      watchOverflow: true,
      autoHeight: true,
      watchSlidesVisibility: true,
      watchSlidesProgress: true,
    });

    const mainSwiper = useSwiper(mainSwiperRef, {
      autoplay: {
        delay: 7000,
        disableOnInteraction: true,
        stopOnLastSlide: false,
      },
      rewind: true,
      watchOverflow: true,
      watchSlidesVisibility: true,
      watchSlidesProgress: true,
      preventInteractionOnTransition: true,
      thumbs: { swiper: thumbsSwiper.instance.value },
      spaceBetween: 25,
      freeMode: false,
    });

    function onVideoPlay() {
      if (mainSwiper.instance.value?.autoplay) {
        mainSwiper.instance.value.autoplay.stop();
      }
    }
    function onVideoPause() {
      if (mainSwiper.instance.value?.autoplay) {
        mainSwiper.instance.value.slideNext();
        mainSwiper.instance.value.autoplay.start();
      }
    }

    function onChange(swiper: any) {
      // If there is no video, do nothing special
      if (!videoCoverUrl.value) return;

      // The video is always the first slide (index 0)
      const videoSlide = swiper.slides[0];
      const videoElement = videoSlide?.querySelector('video');

      if (videoElement) {
        if (swiper.activeIndex === 0) {
          videoElement.currentTime = 0;
          videoElement.pause();
          videoElement.play();
        } else {
          videoElement.pause();
          videoElement.currentTime = 0;
        }
      }
    }

    function onInit(swiper: any) {
      onChange(swiper);
    }

    watch(() => props.activeVariation, (newVal) => {
      if (newVal?.image) {
        const foundIndex = finalGallery.value.findIndex(
          img => img.databaseId === newVal.image?.databaseId
        );
        if (foundIndex !== -1 && mainSwiper.instance.value) {
          mainSwiper.instance.value.slideTo(foundIndex);
        }
      }
    });

    onMounted(() => {
      watch(
        () => [mainSwiper.instance.value, thumbsSwiper.instance.value],
        ([main, thumbs]) => {
          if (main && thumbs) {
            main.on('slideChangeTransitionStart', () => {
              thumbs.slideTo(main.activeIndex);
              onChange(main);
            });

          }
        },
        { immediate: true }
      );
    });
</script>

    <template>
      <div class="flex gap-6">
        <ClientOnly>
          <div v-if="finalGallery.length > 1" class="thumbs-wrapper">
            <swiper-container ref="thumbsSwiperRef" direction="vertical" :space-between="15"
              class="gallery-images-thumbs">
              <swiper-slide v-for="(item, idx) in finalGallery" :key="item.databaseId || idx">
                <template v-if="item.type === 'video'">
                  <img :src="item.poster" :alt="node.name + ' video'" :title="node.name + ' video'"
                    class="cursor-pointer rounded-xl" width="65" loading="lazy" />
                  <span class="thumb-video-play-icon"></span>
                </template>
                <template v-else>
                  <NuxtImg class="cursor-pointer rounded-xl" :width="65" :src="item.sourceUrl"
                    :alt="item.altText || node.name" :title="item.title || node.name" placeholder
                    placeholder-class="blur-xl" loading="lazy" />
                </template>
              </swiper-slide>
            </swiper-container>
          </div>

          <div class="relative rounded-xl overflow-hidden">
            <SaleBadge :node class="absolute text-base top-4 right-4 z-10" />
            <swiper-container ref="mainSwiperRef" :thumbs="{ swiper: thumbsSwiper.instance.value }"
              class="main-gallery-swiper rounded-xl object-contain w-full min-w-[350px]"
               @slidechange="onChange" @swiper="onInit">
              <swiper-slide v-for="(item, idx) in finalGallery" :key="item.databaseId || idx">
                <template v-if="item.type === 'video'">
                  <video autoplay muted playsinline controlsList="nofullscreen nodownload noremoteplayback"
                    :poster="item.poster" class="rounded-xl object-contain w-full h-full"
                    @play="onVideoPlay" @pause="onVideoPause" @ended="onVideoPause">
                    <source :src="item.sourceUrl" type="video/mp4" />
                    Your browser does not support the video tag.
                  </video>
                </template>
                <template v-else>
                  <NuxtImg class="rounded-xl object-contain w-full h-full"
                    :alt="item.altText || node.name" :title="item.title || node.name"
                    :src="item.sourceUrl || FALLBACK_IMG" fetchpriority="high" placeholder placeholder-class="blur-xl"
                    loading="lazy" />
                </template>
              </swiper-slide>
            </swiper-container>
            <button @click="mainSwiper.prev()" class="swiper-button-prev" part="button-prev">
            </button>
            <button @click="mainSwiper.next()" class="swiper-button-next" part="button-next">
            </button>
          </div>
        </ClientOnly>
      </div>

    </template>

<style scoped>

.main-gallery-swiper, .thumbs-wrapper, .gallery-images-thumbs {
  height: 100%;
  max-width: 525px;
}

.swiper-slide-thumb-active {
  transform: scale(0.91);
}

.swiper-button-prev,
.swiper-button-next {
  width: 20px;
  height: 20px; 
  background: rgba(255, 255, 255, 0.5);
  position: absolute;
  top: 70%;
  transform: translateY(-70%);  
  transition: box-shadow 0.3s ease;
  box-shadow: 0 0 10px 5px rgba(194, 194, 194, 0.2); 
}
 
.swiper-button-prev {
  left: 0px;
  border-top-right-radius: 50%;
  border-bottom-right-radius: 50%;
}

.swiper-button-next {
  right: 0px;
  border-bottom-left-radius: 50%;
  border-top-left-radius: 50%;
}

.swiper-button-next:hover,
.swiper-button-prev:hover {
  box-shadow: 0 0 10px 5px rgba(194, 194, 194, 0.5); 
}

/*video icon*/
.thumb-video-play-icon {
  position: absolute;
  top: 50%;
  left: 50%;
  width: 35px;
  height: 35px;
  background: rgba(255, 27, 27, 0.5);
  border-radius: 50%;
  transform: translate(-50%, -50%);
  cursor: pointer;
  display: flex;
  justify-content: center;
  align-items: center;
}

.thumb-video-play-icon::before {
  content: '';
  display: block;
  margin-left: 4px;
  border-style: solid;
  border-width: 7px 0 7px 14px;
  border-color: transparent transparent transparent white;
}

.swiper-slide-thumb-active .thumb-video-play-icon::before {
  content: '';
  display: block;
  width: 14px;
  height: 14px;
  background: white;
  border-radius: 2px;
  margin-left: 0;
  border: none;
}
</style>

