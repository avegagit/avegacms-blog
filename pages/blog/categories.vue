<template>
  <CmsDashboardPage>
    <CmsDashboardPageHeader title="Управление категориями">
      <template #right>
        <CmsButton
          label="Создать категорию"
          title="Создать категорию"
          icon="i-heroicons-plus-16-solid"
          @click="create"
        />
      </template>
    </CmsDashboardPageHeader>
    <CmsDashboardPageContent ref="contentRef">
      <CmsCard :ui="{ body: { padding: 'p-0 sm:p-0' } }">
        <CmsTable v-bind="{ loading, rows, columns }">
          <template #id-data="{ row }: { row: BlogCategory }">
            <a
              href="#"
              class="text-primary p-2 font-medium"
              title="Редактировать"
              @click.prevent="edit(row.id)"
            >
              {{ row.id }}
            </a>
          </template>

          <template #num-data="{ row }: { row: BlogCategory }">
            <NuxtLink
              :to="`/blog/posts?parent=${row.id}`"
              class="text-primary p-2 font-medium"
              title="Перейти к постам"
            >
              {{ row.num }}
            </NuxtLink>
          </template>

          <template #actions-data="{ row }: { row: BlogCategory }">
            <CmsDropdown :items="actions(row)">
              <CmsButton
                color="gray"
                variant="ghost"
                icon="i-heroicons-ellipsis-horizontal-20-solid"
                title="Действия"
              />
            </CmsDropdown>
          </template>

          <template #empty-state>
            <div class="flex flex-col items-center justify-center gap-3 py-6">
              <span class="text-sm italic">Список категорий пуст!</span>
              <CmsButton
                label="Создать категорию"
                title="Создать категорию"
                icon="i-heroicons-plus-16-solid"
                @click="create"
              />
            </div>
          </template>
        </CmsTable>

        <template #footer>
          <div class="flex justify-end">
            <CmsPagination v-bind="pagination" v-model="page" />
          </div>
        </template>
      </CmsCard>
    </CmsDashboardPageContent>
  </CmsDashboardPage>
</template>

<script setup lang="ts">
import {
  CmsConfirmModal,
  BlogCategoriesCreateModal,
  BlogCategoriesEditModal,
} from "#components";
import type { CmsPagination } from "avegacms/src/types/core";
import type { BlogCategory } from "#module/blog/types";
import type { DropdownItem } from "#ui/types";

useSeoMeta({
  title: "Управление категориями",
});

const api = useApi();
const modal = useModal();
const toast = useToast();
const route = useRoute();
const screenSpinner = useScreenSpinner();

const contentRef = ref();

const { data, status, execute } = useAsyncData(() => {
  return api<{ data: BlogCategory[]; meta: { pagination: CmsPagination } }>(
    "admin/blog/category",
    { query: route.query }
  );
});

const loading = computed(() => status.value === "pending");
const rows = computed(() => data.value?.data ?? []);
const columns = [
  {
    key: "id",
    label: "ID",
  },
  {
    key: "title",
    label: "Название",
  },
  {
    key: "num",
    label: "Кол-во постов",
  },
  {
    key: "actions",
  },
];

const actions = (row: BlogCategory) => {
  return [
    [
      {
        label: "Редактировать",
        icon: "i-heroicons-pencil-square-20-solid",
        click: () => edit(row.id),
      },
      {
        label: "Перейти к постам",
        icon: "i-heroicons-list-bullet",
        to: `/blog/posts?parent=${row.id}`,
      },
    ],
    [
      {
        label: "Удалить",
        icon: "i-heroicons-trash-20-solid",
        class: "text-red-500",
        iconClass: "text-red-500",
        click: () => remove(row.id),
      },
    ],
  ] as DropdownItem[][];
};

const page = useRouteQuery("page", 1, { transform: Number });
const pagination = computed(() => {
  return {
    pageCount: data.value?.meta.pagination.limit ?? 0,
    total: data.value?.meta.pagination.total ?? 0,
  };
});

const create = () => {
  modal.open(BlogCategoriesCreateModal, { onCreate: () => execute() });
};
const edit = async (id: number) => {
  screenSpinner.show();

  try {
    const r = await api<{ data: BlogCategory }>(`admin/blog/category/${id}`);
    await screenSpinner.hide();
    modal.open(BlogCategoriesEditModal, {
      category: r.data,
      onUpdate: execute,
    });
  } catch {
    screenSpinner.hide();
    toast.add({
      title: "Не удалось получить данные.",
      color: "red",
    });
  }
};
const remove = (id: number) => {
  const loading = shallowRef(false);
  modal.open(CmsConfirmModal, {
    title: "Внимание!",
    message: "Вы действительно хотите удалить категорию?",
    resolve: { loading },
    onResolve: async () => {
      loading.value = true;

      try {
        await api(`admin/blog/category/${id}`, { method: "DELETE" });
        modal.close();
        toast.add({
          title: "Категория успешно удалена.",
          color: "green",
        });
        execute();
      } catch {
        loading.value = false;
        toast.add({
          title: "Не удалось удалить.",
          color: "red",
        });
      }
    },
    onReject: () => modal.close(),
  });
};

watch(
  () => page.value,
  () => {
    (contentRef.value?.$el as HTMLDivElement).scrollTo({
      top: 0,
      behavior: "smooth",
    });
  }
);
watchDebounced(
  () => route.query,
  () => execute(),
  { debounce: 500, deep: true }
);
</script>

<style scoped></style>
