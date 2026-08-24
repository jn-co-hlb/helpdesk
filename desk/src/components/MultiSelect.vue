<template>
  <!-- HLB-FORK: multi-select — a field the submitter can tick several answers
       on. Upstream's intake form has no such control: `url_method` renders a
       single-choice Autocomplete, and a Select holds one value. Onboarding needs
       both (a joiner goes on several distribution lists and needs a laptop AND a
       monitor), and making them free text would lose the live M365 list.

       Built AROUND the existing Autocomplete rather than replacing it, so the
       search, filtering and keyboard handling stay upstream's and keep working
       after a rebase. This adds only "append on pick" and "chips you can remove".
       See customisations.manifest.json id=ui-multi-select. -->
  <div class="w-full space-y-1.5">
    <Autocomplete
      :options="available"
      :value="null"
      :placeholder="placeholder"
      size="sm"
      @change="add"
    />
    <div v-if="selected.length" class="flex flex-wrap gap-1.5">
      <button
        v-for="value in selected"
        :key="value"
        type="button"
        class="flex items-center gap-1 rounded bg-surface-gray-2 px-2 py-1 text-p-sm text-ink-gray-7 hover:bg-surface-gray-3"
        @click="remove(value)"
      >
        <span>{{ labelFor(value) }}</span>
        <FeatherIcon name="x" class="h-3 w-3" />
      </button>
    </div>
  </div>
</template>

<script setup lang="ts">
import { Autocomplete } from "@/components";
import { FeatherIcon } from "frappe-ui";
import { computed } from "vue";

interface Option {
  label: string;
  value: string;
}

const props = defineProps<{
  options?: Option[];
  value?: string;
  placeholder?: string;
}>();

const emit = defineEmits(["change"]);

// Stored as one string so the column stays Small Text — deliberately NOT a
// Select, because frappe's _validate_selects throws on a value outside the
// options list and a multi-select holds several joined together.
//
// NOTE this makes a comma structural: an option VALUE must not contain one.
// Both current users are safe — distribution lists are email addresses and the
// equipment options use colons — but it is a real constraint on new options.
const SEPARATOR = ", ";

const selected = computed<string[]>(() =>
  (props.value || "")
    .split(",")
    .map((v) => v.trim())
    .filter(Boolean)
);

// Hide what is already picked, so the list cannot produce duplicates.
const available = computed<Option[]>(() =>
  (props.options || []).filter((o) => !selected.value.includes(o.value))
);

function labelFor(value: string): string {
  return (props.options || []).find((o) => o.value === value)?.label || value;
}

function commit(values: string[]): void {
  emit("change", { value: values.join(SEPARATOR) });
}

function add(option: Option | string | null): void {
  const value = typeof option === "string" ? option : option?.value;
  if (!value || selected.value.includes(value)) return;
  commit([...selected.value, value]);
}

function remove(value: string): void {
  commit(selected.value.filter((v) => v !== value));
}
</script>
