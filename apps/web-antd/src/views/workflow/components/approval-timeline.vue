<script setup lang="tsx">
import type { TimelineProps } from 'antdv-next';

import type { Flow } from '#/api/workflow/instance/model';

import { computed, h } from 'vue';

import { Empty, Timeline } from 'antdv-next';

import ApprovalTimelineItem from './approval-timeline-item.vue';

interface Props {
  list: Flow[];
}

const props = defineProps<Props>();

const items = computed<TimelineProps['items']>(() => {
  const { list } = props;
  return list.map((item) => ({
    key: item.id,
    content: h(ApprovalTimelineItem, { item }),
  }));
});
</script>

<template>
  <Timeline v-if="list.length > 0" :items="items" />
  <Empty v-else />
</template>
