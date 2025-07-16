<script lang="ts" setup>
import ReviewsScore from '@/woonuxt_base/app/components/productElements/ReviewsScore.vue';

const props = defineProps({
  product: { type: Object, default: null },
});

const reviews = ref(props.product?.siteReviews?.reviews || []);
const pageInfo = ref(props.product?.siteReviews?.pageInfo || { total: 0, totalPages: 1 });
const currentPage = ref(1);
const selectedRating = ref(0);
const hasContent = ref(true);
const loading = ref(false);
const route = useRoute();
const slug = route.params.slug as string;
const openResponses = ref<{ [id: string]: boolean }>({});
const toggleResponse = (id: string) => {
  openResponses.value[id] = !openResponses.value[id];
};

const averageRating = ref(props.product?.siteReviews?.averageRating || 0);
const ratingsBreakdown = ref(props.product?.siteReviews?.ratingsBreakdown || []);
const totalReviews = ref(props.product?.siteReviews?.totalReviews || 0);

const reviewKey = (review: any) => `${review.author}-${review.date}`;

const fetchReviews = async () => {
  loading.value = true;
  try {
    const { data } = await useAsyncGql('getProductReviews', {
      slug,
      siteReviews_page: currentPage.value,
      siteReviews_perPage: 10,
      siteReviews_rating: selectedRating.value,
      siteReviews_hasContent: hasContent.value,
    });
    const siteReviews = data.value.product?.siteReviews as any;
    reviews.value = siteReviews?.reviews || [];
    pageInfo.value = siteReviews?.pageInfo || { total: 0, totalPages: 1 };
    // Do NOT update summary refs here
  } catch (error) {
    console.error(error);
  } finally {
    loading.value = false;
  }
};

watch([currentPage, selectedRating, hasContent], fetchReviews);
watch([selectedRating, hasContent], () => {
  currentPage.value = 1;
});

const handleSelectRating = (r: number) => {
  selectedRating.value = r;
};

const maxButtons = 5; // Number of page buttons to show around the current page
const paginationButtons = computed(() => {
  const total = pageInfo.value.totalPages;
  const current = currentPage.value;
  if (total <= 7) {
    return Array.from({ length: total }, (_, i) => i + 1);
  }
  const buttons = [];
  if (current > 3) buttons.push(1, '...');
  const start = Math.max(2, current - 1);
  const end = Math.min(total - 1, current + 1);
  for (let i = start; i <= end; i++) buttons.push(i);
  if (current < total - 2) buttons.push('...', total);
  else buttons.push(total);
  return buttons;
});

</script>

<template>
  <div>
    <div class="flex flex-col md:flex-row md:gap-8">
      <!-- Reviews (left, 2/3) -->
      <div class="w-full md:w-2/3 order-2 md:order-1">
        <!-- Removed filter dropdown -->
        <div>
          <div class="divide-y" v-if="reviews && reviews.length">
            <div v-for="review in reviews" :key="review.id" class="my-2 py-8">
              <div class="flex gap-4 items-center">
                <img v-if="review.avatar" :src="review.avatar" class="rounded-full h-12 w-12" />
                <div class="grid gap-1">
                  <div class="text-sm">
                    <span class="font-semibold">{{ review.author }}</span>
                    <span class="italic text-gray-400">
                      – {{ new Date(review.date).toLocaleString($t('messages.general.langCode'), { month: 'long', day: 'numeric', year: 'numeric' }) }}</span
                    >
                  </div>
                  <StarRating :rating="review.rating" :hide-count="true" class="text-sm" />
                </div>
              </div>
              <div class="mt-4 text-gray-700 italic prose-sm" v-html="review.content"></div>
              <div v-if="review.response" class="mt-4">
                <button
                  class="px-3 py-1 border rounded bg-gray-200 hover:bg-gray-300 text-sm font-semibold"
                  @click="toggleResponse(reviewKey(review))"
                >
                  {{ openResponses[reviewKey(review)] ? 'Скрыть ответ' : 'Показать ответ представителя магазина' }}
                </button>
                <div
                  v-if="openResponses[reviewKey(review)]"
                  class="p-4 bg-gray-100 rounded-lg mt-2"
                >
                  <p class="font-semibold">Ответ представителя магазина</p>
                  <div class="text-gray-700 italic prose-sm" v-html="review.response"></div>
                </div>
              </div>
            </div>
          </div>
          <div v-else class="w-full text-center py-8">
            <p>No reviews found.</p>
          </div>
        </div>
        <div class="flex justify-center mt-8 mb-16 col-span-full tabular-nums" v-if="pageInfo.totalPages > 1">
          <nav class="inline-flex self-end -space-x-px rounded-md shadow-sm isolate" aria-label="Pagination">
            <!-- PREV -->
            <button
              @click="currentPage--"
              :disabled="currentPage === 1"
              class="prev"
              :class="{ 'cursor-not-allowed': currentPage === 1 }"
              :aria-disabled="currentPage === 1"
              aria-label="Previous"
            >
              <Icon name="ion:chevron-back-outline" size="20" class="w-5 h-5" />
            </button>
            <!-- NUMBERS (smart with ellipsis) -->
            <template v-for="btn in paginationButtons" :key="btn + '-' + currentPage">
              <button
                v-if="btn !== '...'"
                @click="currentPage = btn"
                :aria-current="btn === currentPage ? 'page' : undefined"
                class="page-number"
                :class="{ 'bg-primary border-primary border bg-opacity-10 text-primary z-10': btn === currentPage }"
              >
                {{ btn }}
              </button>
              <span v-else class="px-2 text-gray-400 select-none">...</span>
            </template>
            <!-- NEXT -->
            <button
              @click="currentPage++"
              :disabled="currentPage === pageInfo.totalPages"
              class="next"
              :class="{ 'cursor-not-allowed': currentPage === pageInfo.totalPages }"
              :aria-disabled="currentPage === pageInfo.totalPages"
              aria-label="Next"
            >
              <Icon name="ion:chevron-forward-outline" size="20" class="w-5 h-5" />
            </button>
          </nav>
        </div>
      </div>
      <!-- Summary (right, 1/3) -->
      <div class="w-full md:w-1/3 mb-8 md:mb-0 order-1 md:order-2">
        <ReviewsScore
          :average-rating="averageRating"
          :ratings-breakdown="ratingsBreakdown"
          :total-reviews="totalReviews"
          :selected-rating="selectedRating"
          @select-rating="handleSelectRating"
        />
        <div class="flex items-center gap-2 mt-4 mb-6">
          <input type="checkbox" v-model="hasContent" id="hasContent" />
          <label for="hasContent" class="text-sm">Has Content</label>
        </div>
      </div>
    </div>
  </div>
</template>

<style lang="postcss" scoped>
.prev,
.next,
.page-number {
  @apply bg-white border font-medium border-gray-300 text-sm p-2 text-gray-500 relative inline-flex items-center hover:bg-gray-50 focus:z-10;
}

.prev {
  @apply rounded-l-md;
}

.next {
  @apply rounded-r-md;
}

.page-number {
  @apply px-3;
}

.page-number[aria-current='page'] {
  @apply bg-primary border-primary border bg-opacity-10 text-primary z-10;
}
</style>
