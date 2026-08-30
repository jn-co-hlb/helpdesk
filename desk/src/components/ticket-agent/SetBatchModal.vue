<!-- HLB-FORK: ticket-batch — put a page of selected tickets in one named batch.

     One announcement goes out and twenty people reply with twenty different
     problems. Merging them is wrong — each has its own requester, its own SLA
     clock and needs its own answer — but they still want to be findable as one
     lot afterwards.

     Not frappe tags: Tag / Tag Link exist and helpdesk's ticket API even
     returns a ticket's tags, but nothing in the agent UI renders, edits or
     filters them, and `_user_tags` is neither a DocField nor a Custom Field so
     `get_filterable_fields` cannot see it either. A Link Custom Field is
     filterable, sortable, groupable and columnable in this list on day one,
     which is the whole point of grouping them.
     See customisations.manifest.json id=ui-ticket-batch. -->
<template>
  <Dialog
    v-model="open"
    :options="{
      title: __('Set batch'),
      size: 'md',
    }"
  >
    <template #body-content>
      <div class="flex flex-col gap-3">
        <p class="text-p-sm text-ink-gray-6">
          {{
            __(
              "Group these tickets under one name so they can be found together later. Clearing the batch removes them from it."
            )
          }}
        </p>
        <Link
          class="w-full"
          doctype="HLB Ticket Batch"
          :modelValue="batch"
          :filters="{ closed: 0 }"
          :placeholder="__('Search or create a batch')"
          @update:modelValue="(v: string) => (batch = v)"
          @create="handleCreate"
        />
      </div>
    </template>
    <template #actions>
      <div class="flex items-center justify-end gap-2">
        <Button :label="__('Cancel')" @click="open = false" />
        <Button
          variant="solid"
          :loading="setBatch.loading"
          :label="
            props.selections.size === 1
              ? __('Set batch')
              : __('Set batch on {0} tickets', String(props.selections.size))
          "
          @click="handleSubmit"
        />
      </div>
    </template>
  </Dialog>
</template>

<script setup lang="ts">
import { Link } from "@/components";
import { __ } from "@/translation";
import { call, createResource, Dialog, toast } from "frappe-ui";
import { ref } from "vue";

const props = defineProps<{ selections: Set<string> }>();
const emit = defineEmits<{ (e: "success"): void }>();

const open = defineModel<boolean>();
const batch = ref("");

const setBatch = createResource({
  url: "company_helpdesk.api.set_batch",
  makeParams: () => ({
    names: Array.from(props.selections),
    batch: batch.value,
  }),
  onSuccess: (changed: number) => {
    // Report what actually changed, not what was asked for: set_batch skips
    // tickets the agent cannot write and tickets already in this batch, and
    // "12 of 20" is the useful sentence when a team restriction is in play.
    toast.success(
      changed === props.selections.size
        ? __("{0} ticket(s) updated", [String(changed)])
        : __("{0} of {1} ticket(s) updated", [
            String(changed),
            String(props.selections.size),
          ])
    );
    open.value = false;
    batch.value = "";
    emit("success");
  },
  onError: (error: any) => {
    toast.error(error?.messages?.[0] || __("Could not set the batch"));
  },
});

/**
 * "Create New" in the Link dropdown. A batch is a name and nothing else at the
 * point of creation, so making the agent leave the list to make one would be
 * the reason nobody uses this.
 */
async function handleCreate(value: string, close: Function) {
  const name = (value || "").trim();
  if (!name) return;
  try {
    const doc = await call("frappe.client.insert", {
      doc: { doctype: "HLB Ticket Batch", batch_name: name },
    });
    batch.value = doc.name;
    close();
  } catch (error: any) {
    toast.error(error?.messages?.[0] || __("Could not create the batch"));
  }
}

function handleSubmit() {
  if (!props.selections.size) return;
  setBatch.submit();
}
</script>
