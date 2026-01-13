<script setup lang="tsx">
import type { DescriptionsProps } from 'antdv-next';

import type { LeaveVO } from '../leave/api/model';

import { computed, onMounted, shallowRef } from 'vue';

import { Descriptions, Skeleton } from 'antdv-next';
import dayjs from 'dayjs';

import { leaveInfo } from './api';
import { leaveTypeOptions } from './data';

defineOptions({
  name: 'LeaveDescription',
  inheritAttrs: false,
});

const props = defineProps<{ businessId: number | string }>();

const data = shallowRef<LeaveVO>();
onMounted(async () => {
  const resp = await leaveInfo(props.businessId);
  data.value = resp;
});

const leaveType = computed(() => {
  return (
    leaveTypeOptions.find((item) => item.value === data.value?.leaveType)
      ?.label ?? '未知'
  );
});

function formatDate(date: string) {
  return dayjs(date).format('YYYY-MM-DD');
}

const items = computed<DescriptionsProps['items']>(() => {
  if (!data.value) {
    return [];
  }
  const info = data.value;
  return [
    {
      content: leaveType.value,
      label: '请假类型',
    },
    {
      content: `${formatDate(info.startDate)} - ${formatDate(info.endDate)}`,
      label: '请假时间',
    },
    {
      content: `${info.leaveDays}天`,
      label: '请假时长',
    },
    {
      content: info.remark || '无',
      label: '请假原因',
    },
  ];
});
</script>

<template>
  <div class="rounded-[6px] border p-2">
    <Descriptions v-if="data" :column="1" :items="items" size="middle" />

    <Skeleton v-else active />
  </div>
</template>
