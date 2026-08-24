<template>
  <div class="space-y-1.5" v-if="field.display_via_depends_on">
    <span class="block text-sm text-ink-gray-7">
      {{ field.label }}
      <span v-if="field.required" class="place-self-center text-ink-red-6">
        *
      </span>
    </span>
    <div class="flex gap-2 items-center [&>div]:flex-1">
      <component
        class="w-full"
        :is="component"
        :placeholder="placeholder"
        :value="transValue"
        :disabled="field.disabled"
        :model-value="transValue"
        @update:model-value="emitUpdate(field.fieldname, $event)"
        @change="
          emitUpdate(
            field.fieldname,
            $event.target?.value || $event.value || $event
          )
        "
      />
      <slot name="label-extra" />
    </div>
    <!-- HLB-FORK: field-notes — render a Custom Field's `description` as helper
         text under the input. Upstream renders only the label and the control,
         so there was no way to put per-field guidance on the intake form; the
         only note mechanism was HD Ticket Template.about, which is per-template
         and therefore shows for every ticket type. Because our custom fields are
         already depends_on-gated by type, a description here gives us per-type
         notes with no extra machinery.
         See customisations.manifest.json id=ui-field-notes. -->
    <p v-if="field.description" class="text-p-sm text-ink-gray-5">
      {{ __(field.description) }}
    </p>
  </div>
</template>

<script setup lang="ts">
import { Autocomplete, Link, MultiSelect } from "@/components";
import { APIOptions, Field } from "@/types";
import { parseApiOptions } from "@/utils";
import {
  createResource,
  DatePicker,
  DateTimePicker,
  FormControl,
} from "frappe-ui";
import { computed, h } from "vue";

type Value = string | number | boolean;

interface P {
  field: Field;
  value: Value;
}

interface R {
  fieldname: Field["fieldname"];
  value: Value;
}

interface E {
  (event: "change", value: R);
}

const props = defineProps<P>();
const emit = defineEmits<E>();

const component = computed(() => {
  // HLB-FORK: multi-select — checked FIRST so it wins for a field that also has
  // a url_method: the distribution-list picker is both live-loaded AND
  // multi-valued, and the url_method branch below would otherwise claim it and
  // render a single-choice control.
  // See customisations.manifest.json id=ui-multi-select.
  if (props.field.hlb_multiple) {
    return h(MultiSelect, {
      options: props.field.url_method
        ? apiOptions.data
        : props.field.options
          ? props.field.options
              .split("\n")
              .filter(Boolean)
              .map((o) => ({ label: o, value: o }))
          : [],
    });
  } else if (props.field.url_method) {
    return h(Autocomplete, {
      options: apiOptions.data,
      size: "sm",
    });
  } else if (props.field.fieldtype === "Link" && props.field.options) {
    return h(Link, {
      doctype: props.field.options,
      filters: props.field.filters,
      pageLength: 999,
    });
  } else if (props.field.fieldtype === "Select") {
    return h(Autocomplete, {
      options: props.field.options
        ? props.field.options.split("\n").map((o) => ({ label: o, value: o }))
        : [],
      size: "sm",
    });
  } else if (props.field.fieldtype === "Check") {
    return h(Autocomplete, {
      options: [
        {
          label: "Yes",
          value: 1,
        },
        {
          label: "No",
          value: 0,
        },
      ],
      size: "sm",
    });
  } else if (props.field.fieldtype === "Datetime") {
    return h(DateTimePicker, {
      format: `${window.date_format.toUpperCase()} ${window.time_format}`,
    });
  } else if (props.field.fieldtype === "Date") {
    return h(DatePicker, {
      id: props.field.fieldname,
      format: window.date_format.toUpperCase(),
    });
  } else {
    return h(FormControl, {
      debounce: 500,
    });
  }
});

const apiOptions = createResource({
  url: props.field.url_method,
  auto: !!props.field.url_method,
  transform: (data: APIOptions) => {
    return parseApiOptions(data);
  },
});

const transValue = computed(() => {
  if (props.field.fieldtype === "Check") {
    return props.value ? "Yes" : "No";
  }
  return props.value;
});

const placeholder = computed(() => {
  if (props.field.placeholder) {
    return props.field.placeholder;
  }
  if (props.field.fieldtype === "Data" && !props.field.url_method) {
    return "Type something";
  } else if (
    props.field.fieldtype === "Select" ||
    props.field.fieldtype === "Link"
  ) {
    return "Select an option";
  }
  return "Type something";
});

function emitUpdate(fieldname: Field["fieldname"], value: Value) {
  emit("change", { fieldname, value });
}
</script>
