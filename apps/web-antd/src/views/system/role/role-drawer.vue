<!--
TODO: 这个页面要优化逻辑
-->
<script setup lang="ts">
import type { MenuOption } from '#/api/system/menu/model';

import { computed, nextTick, ref } from 'vue';

import { useVbenDrawer } from '@vben/common-ui';
import { $t } from '@vben/locales';
import { cloneDeep, eachTree } from '@vben/utils';

import { useVbenForm } from '#/adapter/form';
import { menuTreeSelect, roleMenuTreeSelect } from '#/api/system/menu';
import { roleAdd, roleInfo, roleUpdate } from '#/api/system/role';
import { MenuSelectTable } from '#/components/tree';
import { defaultFormValueGetter, useBeforeCloseDiff } from '#/utils/popup';

import { drawerSchema } from './data';

const emit = defineEmits<{ reload: [] }>();

const isUpdate = ref(false);
const title = computed(() => {
  return isUpdate.value ? $t('pages.common.edit') : $t('pages.common.add');
});

const [BasicForm, formApi] = useVbenForm({
  commonConfig: {
    componentProps: {
      class: 'w-full',
    },
    formItemClass: 'col-span-1',
  },
  layout: 'vertical',
  schema: drawerSchema(),
  showDefaultActions: false,
  wrapperClass: 'grid-cols-2 gap-x-4',
});

const menuTree = ref<MenuOption[]>([]);
async function loadMenuTree(
  menus: MenuOption[],
  checkedKeys: (number | string)[],
) {
  eachTree(menus, (node) => {
    node.label = $t(node.label);
  });
  menuTree.value = menus;
  // keys依赖于menu 需要先加载menu
  await nextTick();
  await formApi.setFieldValue('menuIds', checkedKeys);
}

async function customFormValueGetter() {
  const v = await defaultFormValueGetter(formApi)();
  // 获取勾选信息
  const menuIds = menuSelectRef.value?.getCheckedKeys?.() ?? [];
  const mixStr = v + menuIds.join(',');
  return mixStr;
}

const { onBeforeClose, markInitialized, resetInitialized } = useBeforeCloseDiff(
  {
    initializedGetter: customFormValueGetter,
    currentGetter: customFormValueGetter,
  },
);

const [BasicDrawer, drawerApi] = useVbenDrawer({
  onBeforeClose,
  onClosed: handleClosed,
  onConfirm: handleConfirm,
  destroyOnClose: true,
  async onOpenChange(isOpen) {
    if (!isOpen) {
      return null;
    }
    drawerApi.drawerLoading(true);

    const { id } = drawerApi.getData() as { id?: number | string };
    isUpdate.value = !!id;

    if (isUpdate.value && id) {
      // 并行请求角色信息和菜单树
      const [record, menuResp] = await Promise.all([
        roleInfo(id),
        roleMenuTreeSelect(id),
      ]);
      // 先设置表单值(menuCheckStrictly会影响菜单树的节点关联模式)
      await formApi.setValues(record);
      // 再加载菜单树
      await loadMenuTree(menuResp.menus, menuResp.checkedKeys);
    } else {
      const menus = await menuTreeSelect();
      await loadMenuTree(menus, []);
    }
    await markInitialized();

    drawerApi.drawerLoading(false);
  },
});

const menuSelectRef = ref<InstanceType<typeof MenuSelectTable>>();
async function handleConfirm() {
  try {
    drawerApi.lock(true);

    const { valid } = await formApi.validate();
    if (!valid) {
      return;
    }
    // 这个用于提交
    const menuIds = menuSelectRef.value?.getCheckedKeys?.() ?? [];
    // formApi.getValues拿到的是一个readonly对象，不能直接修改，需要cloneDeep
    const data = cloneDeep(await formApi.getValues());
    data.menuIds = menuIds;
    await (isUpdate.value ? roleUpdate(data) : roleAdd(data));
    emit('reload');
    resetInitialized();
    drawerApi.close();
  } catch (error) {
    console.error(error);
  } finally {
    drawerApi.lock(false);
  }
}

async function handleClosed() {
  await formApi.resetForm();
  resetInitialized();
}

/**
 * 通过回调更新 无法通过v-model
 * @param value 菜单选择是否严格模式
 */
function handleMenuCheckStrictlyChange(value: boolean) {
  formApi.setFieldValue('menuCheckStrictly', value);
}
</script>

<template>
  <BasicDrawer :title="title" class="w-[800px]">
    <BasicForm>
      <template #menuIds="slotProps">
        <div class="h-[600px] w-full">
          <!-- association为readonly 不能通过v-model绑定 -->
          <MenuSelectTable
            ref="menuSelectRef"
            :checked-keys="slotProps.value"
            :association="formApi.form.values.menuCheckStrictly"
            :menus="menuTree"
            @update:association="handleMenuCheckStrictlyChange"
          />
        </div>
      </template>
    </BasicForm>
  </BasicDrawer>
</template>
