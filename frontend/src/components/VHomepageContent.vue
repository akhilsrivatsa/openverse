<template>
  <div :id="skipToContentTargetId" tabindex="-1">
    <h1
      class="mb-2 mt-auto text-[40px] font-light leading-tight lg:text-[63px]"
    >
      {{ $t("hero.subtitle") }}
    </h1>

    <p class="text-base leading-relaxed">
      {{ $t("hero.description") }}
    </p>

    <VStandaloneSearchBar
      ref="searchBarRef"
      class="mt-4 md:mt-6"
      :has-popover="!!triggerElement && isContentSwitcherVisible"
      @submit="handleSearch"
    >
      <VSearchTypeButton
        id="search-type-button"
        ref="searchTypeButtonRef"
        class="ms-2"
        v-bind="{ ...triggerA11yProps, ...searchTypeProps }"
        :show-label="isSm"
        aria-controls="content-switcher-popover"
        @click="onTriggerClick"
      />
      <template v-if="triggerElement">
        <VPopoverContent
          v-if="isLg"
          z-index="popover"
          :hide="closeContentSwitcher"
          :trap-focus="false"
          :visible="isContentSwitcherVisible"
          :trigger-element="triggerElement"
          width="w-66"
          aria-labelledby="search-type-button"
        >
          <VSearchTypes
            size="small"
            :use-links="false"
            @select="handleSelect"
          />
        </VPopoverContent>

        <VContentSettingsModalContent
          v-else
          aria-labelledby="search-type-button"
          :close="closeContentSwitcher"
          :visible="isContentSwitcherVisible"
          :use-links="false"
          :show-filters="false"
          @open="openContentSwitcher"
          @select="handleSelect"
        />
      </template>
    </VStandaloneSearchBar>

    <!-- Disclaimer for large screens -->
    <i18n path="hero.disclaimer.content" tag="p" class="mt-4 text-sr">
      <template #openverse>Openverse</template>
      <template #license>
        <VLink
          href="https://creativecommons.org/licenses/"
          class="text-dark-charcoal underline hover:text-dark-charcoal"
          >{{ $t("hero.disclaimer.license") }}</VLink
        >
      </template>
    </i18n>
  </div>
</template>
<script lang="ts">
import { computed, defineComponent, ref, PropType } from "vue"

import type { SearchType } from "~/constants/media"
import { skipToContentTargetId } from "~/constants/window"
import { ensureFocus } from "~/utils/reakit-utils/focus"

import useSearchType from "~/composables/use-search-type"
import { useDialogControl } from "~/composables/use-dialog-control"

import { useUiStore } from "~/stores/ui"

import VContentSettingsModalContent from "~/components/VHeader/VHeaderMobile/VContentSettingsModalContent.vue"
import VLink from "~/components/VLink.vue"
import VPopoverContent from "~/components/VPopover/VPopoverContent.vue"
import VSearchTypeButton from "~/components/VContentSwitcher/VSearchTypeButton.vue"
import VSearchTypes from "~/components/VContentSwitcher/VSearchTypes.vue"
import VStandaloneSearchBar from "~/components/VHeader/VSearchBar/VStandaloneSearchBar.vue"

export default defineComponent({
  name: "VHomepageContent",
  components: {
    VContentSettingsModalContent,
    VSearchTypes,
    VPopoverContent,
    VSearchTypeButton,
    VStandaloneSearchBar,
    VLink,
  },
  props: {
    handleSearch: {
      type: Function as PropType<(query: string) => void>,
      required: true,
    },
    searchType: {
      type: String as PropType<SearchType>,
      required: true,
    },
    setSearchType: {
      type: Function as PropType<(searchType: SearchType) => void>,
      required: true,
    },
  },
  setup(props, { emit }) {
    const searchTypeButtonRef = ref<{ $el: HTMLElement } | null>(null)
    const searchBarRef = ref<{ $el: HTMLElement } | null>(null)
    const nodeRef = computed(() => searchBarRef.value?.$el ?? null)

    const { getSearchTypeProps } = useSearchType()
    const uiStore = useUiStore()

    const searchTypeProps = computed(() => getSearchTypeProps())

    const isContentSwitcherVisible = ref(false)

    const isSm = computed(() => uiStore.isBreakpoint("sm"))
    const isLg = computed(() => uiStore.isBreakpoint("lg"))

    const triggerElement = computed(
      () => searchTypeButtonRef.value?.$el || null
    )

    const lockBodyScroll = computed(() => !isLg.value)

    /**
     * When a search type is selected, we close the popover or modal,
     * and focus the search input.
     *
     * @param searchType
     */
    const handleSelect = (searchType: SearchType) => {
      props.setSearchType(searchType)
      const searchInput = searchBarRef.value?.$el.getElementsByTagName("input")
      if (searchInput) {
        ensureFocus(searchInput[0])
      }
      closeContentSwitcher()
    }

    const {
      close: closeContentSwitcher,
      open: openContentSwitcher,
      onTriggerClick,
      triggerA11yProps,
    } = useDialogControl({
      visibleRef: isContentSwitcherVisible,
      nodeRef,
      lockBodyScroll,
      emit,
    })

    return {
      searchTypeButtonRef,
      searchBarRef,

      isLg,
      isSm,

      triggerElement,
      onTriggerClick,
      handleSelect,
      searchTypeProps,
      closeContentSwitcher,
      openContentSwitcher,
      isContentSwitcherVisible,
      triggerA11yProps,

      skipToContentTargetId,
    }
  },
})
</script>
