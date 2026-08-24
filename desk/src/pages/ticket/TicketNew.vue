<template>
  <div class="flex flex-col overflow-y-auto">
    <LayoutHeader>
      <template #left-header>
        <Breadcrumbs :items="breadcrumbs" />
      </template>
      <template #right-header>
        <CustomActions
          v-if="template.data?._customActions"
          :actions="template.data?._customActions"
        />
      </template>
    </LayoutHeader>
    <!-- Container -->
    <div
      class="flex flex-col gap-5 py-6 h-full flex-1 self-center overflow-auto mx-auto w-full max-w-4xl px-5"
    >
      <!-- custom fields descriptions -->
      <div v-if="Boolean(template.data?.about)" class="">
        <div class="prose-f" v-html="sanitize(template.data.about)" />
      </div>
      <!-- custom fields -->
      <div
        class="grid grid-cols-1 gap-4 sm:grid-cols-3"
        v-if="Boolean(visibleFields)"
      >
        <UniInput
          v-for="field in visibleFields"
          :key="field.fieldname"
          :field="field"
          :value="templateFields[field.fieldname]"
          @change="
            (e) => handleOnFieldChange(e, field.fieldname, field.fieldtype)
          "
        >
          <template v-if="field.fieldname === 'priority'" #label-extra>
            <template
              v-if="
                ticketPriorityResource.dataMap[templateFields[field.fieldname]]
                  ?.description
              "
            >
              <Tooltip
                :text="
                  ticketPriorityResource.dataMap[
                    templateFields[field.fieldname]
                  ].description.trim()
                "
              >
                <lucide-circle-question-mark class="h-4 w-4 text-ink-gray-6" />
              </Tooltip>
            </template>
          </template>
        </UniInput>
      </div>
      <!-- HLB-FORK: type-notice — show the selected HD Ticket Type's own
           `description` as a visible notice. Upstream has no per-type guidance
           anywhere on the intake form: `HD Ticket Template.about` is per
           TEMPLATE so it shows the same words for every type, and the
           field-notes mechanism renders a FIELD's description, which is static.
           Picking the wrong ticket type is our most common intake mistake and
           it mis-routes the ticket, so the guidance has to react to the choice.
           Deliberately a notice and not a Tooltip like the priority one beside
           it: somebody choosing the wrong type does not know they have anything
           to hover over. See customisations.manifest.json id=ui-type-notice. -->
      <div
        v-if="ticketTypeNotice"
        class="rounded border border-outline-gray-2 bg-surface-gray-1 px-3 py-2 text-p-sm text-ink-gray-6"
      >
        {{ ticketTypeNotice }}
      </div>
      <!-- existing fields -->
      <div
        class="flex flex-col"
        :class="(subject.length >= 2 || description.length) && 'gap-5'"
      >
        <div class="flex flex-col gap-2">
          <!-- HLB-FORK: subject-per-type — the label and placeholder come from
               subjectCopy instead of being hardcoded, so a ticket type can
               repurpose Subject. See the comment on SUBJECT_OVERRIDES below. -->
          <span class="block text-sm text-ink-gray-7">
            {{ subjectCopy.label }}
            <span class="place-self-center text-ink-red-5"> * </span>
          </span>
          <FormControl
            v-model="subject"
            type="text"
            :placeholder="subjectCopy.placeholder"
            maxlength="140"
          />
        </div>
        <SearchArticles
          v-if="isCustomerPortal"
          :query="subject"
          class="shadow"
        />
        <div v-if="isCustomerPortal">
          <h4
            v-show="subject.length <= 2 && description.length === 0"
            class="text-p-sm text-ink-gray-4 ml-1"
          >
            {{ __("Please enter a subject to continue") }}
          </h4>
          <TicketTextEditor
            v-show="subject.length > 2 || description.length > 0"
            ref="editor"
            v-model:attachments="attachments"
            v-model:content="description"
            :placeholder="editorHint"
            expand
            :uploadFunction="(file:any)=>uploadFunction(file)"
          >
            <template #bottom-right>
              <Button
                :label="__('Submit')"
                theme="gray"
                variant="solid"
                :disabled="
                  (bodyRequired && $refs.editor?.editor?.isEmpty) ||
                  ticket.loading ||
                  !subject
                "
                @click="() => ticket.submit()"
              />
            </template>
          </TicketTextEditor>
        </div>
      </div>

      <!-- for agent portal -->
      <div v-if="!isCustomerPortal">
        <TicketTextEditor
          ref="editor"
          v-model:attachments="attachments"
          v-model:content="description"
          :placeholder="editorHint"
          expand
          :uploadFunction="(file:any)=>uploadFunction(file)"
        >
          <template #bottom-right>
            <Button
              :label="__('Submit')"
              theme="gray"
              variant="solid"
              :disabled="
                (bodyRequired && $refs.editor?.editor?.isEmpty) ||
                ticket.loading ||
                !subject
              "
              @click="() => ticket.submit()"
            />
          </template>
        </TicketTextEditor>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { LayoutHeader, UniInput } from "@/components";
import {
  handleLinkFieldUpdate,
  handleSelectFieldUpdate,
  parseField,
  setupCustomizations,
} from "@/composables/formCustomisation";
import { useAuthStore } from "@/stores/auth";
import { globalStore } from "@/stores/globalStore";
import { capture } from "@/telemetry";
import { __ } from "@/translation";
import { Field } from "@/types";
import { isCustomerPortal, uploadFunction } from "@/utils";
import {
  Breadcrumbs,
  Button,
  call,
  createListResource,
  createResource,
  FormControl,
  usePageMeta,
} from "frappe-ui";
import { useOnboarding } from "frappe-ui/frappe";
import sanitizeHtml from "sanitize-html";
import {
  computed,
  defineAsyncComponent,
  onMounted,
  reactive,
  ref,
  watch,
} from "vue";
import { useRoute, useRouter } from "vue-router";
import SearchArticles from "../../components/SearchArticles.vue";
const TicketTextEditor = defineAsyncComponent(
  () => import("./TicketTextEditor.vue")
);

interface P {
  templateId?: string;
}

const props = withDefaults(defineProps<P>(), {
  templateId: "",
});

const route = useRoute();
const router = useRouter();
const { $dialog } = globalStore();
const { updateOnboardingStep } = useOnboarding("helpdesk");
const { isManager, userId: userID } = useAuthStore();

const subject = ref("");
const description = ref("");
const attachments = ref([]);
const templateFields = reactive({});

// HLB-FORK: subject-per-type — let a ticket type rename Subject.
//
// WHY: there is exactly ONE new-ticket form for every ticket type (template
// "Default", with per-type fields shown via depends_on), and Subject's label and
// placeholder were hardcoded here. Software Requests needed Subject to BE the
// software name — so the request is readable in the ticket list — without a
// second "Software / tool name" field duplicating it, and without renaming
// Subject for Operational Incidents, HR Onboarding and the rest.
//
// Keyed on the HD Ticket Type name, so adding a type here is a one-line change.
// Anything not listed keeps the upstream wording untouched.
//
// Strings are stored raw and passed through __() at read time, not at module
// load, so translations resolve after the locale is ready.
// HLB-FORK: editor-hint — per-SUB-TYPE placeholder for the body editor.
// Retiring narrative custom fields ("Alternatives considered", "What do you need
// added or changed?") moved that prose into the body, so the body has to ask for
// it. Upstream's placeholder is the single word-pair "Detailed explanation" for
// every ticket in the system.
//
// Keyed on the sub-type VALUE rather than on (type, field): those values are
// already unique across the whole form — they are what the SLA conditions and
// the routing table match on — so a flat map needs no second lookup and stays
// readable as more sub-types arrive.
// See customisations.manifest.json id=ui-editor-hint.
const EDITOR_HINTS: Record<string, string> = {
  "New software request":
    "Please provide as much detail as possible and upload the business case below.",
  Onboarding:
    "Please include: job title, certifications (e.g. CA(SA), RA, CISA), mobile number, and any special requests.",
  Offboarding:
    "Provide specific instructions e.g. Email forwarding address.",
};


// The sub-type lives in a different Custom Field per ticket type, so look
// through the ones that carry one rather than hardcoding a single fieldname.
const SUBTYPE_FIELDS = [
  "sr_category",
  "oi_category",
  "si_category",
  "swr_category",
  "pi_category",
  "au_support_type",
  "hr_subtype",
];

// HLB-FORK: optional-body — sub-types where an empty body is a legitimate
// answer. Upstream disables Submit whenever the editor is empty, which is right
// almost everywhere: a ticket with no description is a ticket somebody has to
// chase. Offboarding is the exception — "remove this person's access on this
// date" is complete once the name and the date are filled in, and forcing a
// sentence there just trains people to type "n/a".
// See customisations.manifest.json id=ui-optional-body.
const OPTIONAL_BODY = new Set(["Offboarding"]);

const bodyRequired = computed(() => {
  const fields = templateFields as Record<string, string>;
  return !SUBTYPE_FIELDS.some((fieldname) => OPTIONAL_BODY.has(fields[fieldname]));
});

const editorHint = computed(() => {
  const fields = templateFields as Record<string, string>;
  for (const fieldname of SUBTYPE_FIELDS) {
    const hint = EDITOR_HINTS[fields[fieldname]];
    if (hint) return __(hint);
  }
  return __("Detailed explanation");
});

const SUBJECT_OVERRIDES: Record<string, { label: string; placeholder: string }> =
  {
    "Software Request": {
      label: "Software tool name",
      placeholder: "Full software name and version if applicable.",
    },
    "HR Onboarding / Offboarding": {
      label: "Ticket name",
      placeholder: "Filled in from the sub-type and full name.",
    },
  };

// HLB-FORK: hr-ticket-name — derive the subject for HR tickets.
// Every onboarding ticket wants the same subject in the same shape, and asking
// a person to type "Onboarding: Jane Smith" by hand guarantees a list where
// half say "New starter" and half say "onboarding jane". Composing it from the
// two fields that already carry the answer makes the queue sortable and the
// ticket findable by name.
//
// It stays editable rather than read-only: the derived value is right almost
// always, and the rare case that needs something else should not need an agent.
// See customisations.manifest.json id=ui-hr-ticket-name.
const HR_TICKET_TYPE = "HR Onboarding / Offboarding";

watch(
  () => {
    const fields = templateFields as Record<string, string>;
    return [fields["ticket_type"], fields["hr_subtype"], fields["hr_full_name"]];
  },
  ([ticketType, subtype, fullName]) => {
    if (ticketType !== HR_TICKET_TYPE) return;
    const name = (fullName || "").trim();
    if (!subtype || !name) return;
    subject.value = `${subtype}: ${name}`;
  }
);

const subjectCopy = computed(() => {
  const ticketType = (templateFields as Record<string, string>)["ticket_type"];
  const override = SUBJECT_OVERRIDES[ticketType];
  return {
    label: __(override?.label ?? "Subject"),
    placeholder: __(override?.placeholder ?? "A short description"),
  };
});

const template = createResource({
  url: "helpdesk.helpdesk.doctype.hd_ticket_template.api.get_one",
  makeParams: () => ({
    name: props.templateId || "Default",
  }),
  auto: true,
  onSuccess: (data) => {
    description.value = data.description_template || "";
    oldFields = window.structuredClone(data.fields || []);
    setupCustomizations(template, {
      doc: templateFields,
      call,
      router,
      $dialog,
      applyFilters,
    });
    setupTemplateFields(data.fields);
  },
});

function setupTemplateFields(fields) {
  fields.forEach((field: Field) => {
    templateFields[field.fieldname] = "";
  });
}

const ticketPriorityResource = createListResource({
  doctype: "HD Ticket Priority",
  fields: ["name", "description"],
  auto: true,
  cache: "ticketPriorities",
});

// HLB-FORK: type-notice — mirrors ticketPriorityResource above, including the
// dataMap lookup, so the two behave the same way and the portal already proves
// this list is readable to a Website User.
const ticketTypeResource = createListResource({
  doctype: "HD Ticket Type",
  fields: ["name", "description"],
  auto: true,
  cache: "ticketTypes",
});

const ticketTypeNotice = computed(() => {
  const selected = (templateFields as Record<string, string>)["ticket_type"];
  if (!selected) return "";
  return ticketTypeResource.dataMap?.[selected]?.description?.trim() || "";
});

let oldFields = [];

function applyFilters(fieldname: string, filters: any = null) {
  const f: Field = template.data.fields.find((f) => f.fieldname === fieldname);
  if (!f) return;
  if (f.fieldtype === "Select") {
    handleSelectFieldUpdate(f, fieldname, filters, templateFields, oldFields);
  } else if (f.fieldtype === "Link") {
    handleLinkFieldUpdate(f, fieldname, filters, templateFields, oldFields);
  }
}

const customOnChange = computed(() => template.data?._customOnChange);

const visibleFields = computed(() => {
  let _fields = template.data?.fields?.filter(
    (f) => !isCustomerPortal.value || !f.hide_from_customer
  );
  if (!_fields) return [];
  return _fields.map((field) => parseField(field, templateFields));
});

function handleOnFieldChange(e: any, fieldname: string, fieldtype: string) {
  templateFields[fieldname] = e.value;
  const fieldDependentFns = customOnChange.value?.[fieldname];
  if (fieldDependentFns) {
    fieldDependentFns.forEach((fn: Function) => {
      fn(e.value, fieldtype);
    });
  }
}

const ticket = createResource({
  url: "helpdesk.helpdesk.doctype.hd_ticket.api.new",
  debounce: 300,
  makeParams: () => ({
    doc: {
      description: description.value,
      subject: subject.value,
      template: props.templateId,
      ...templateFields,
    },
    attachments: attachments.value,
  }),
  validate: (params) => {
    const fields = visibleFields.value?.filter((f) => f.required) || [];
    const toVerify = [...fields, "subject", "description"];
    for (const field of toVerify) {
      if (!params.doc[field.fieldname || field]) {
        return `${field.label || field} is required`;
      }
    }
  },
  onSuccess: (data) => {
    router.push({
      name: isCustomerPortal.value ? "TicketCustomer" : "TicketAgent",
      params: {
        ticketId: data.name,
      },
    });
    if (isManager) {
      updateOnboardingStep("create_first_ticket", true, false, () =>
        localStorage.setItem("firstTicket", data.name)
      );
    }
  },
});

function sanitize(html: string) {
  return sanitizeHtml(html, {
    allowedTags: sanitizeHtml.defaults.allowedTags.concat(["img"]),
  });
}

const breadcrumbs = computed(() => {
  const items = [
    {
      label: __("Tickets"),
      route: {
        name: isCustomerPortal.value ? "TicketsCustomer" : "TicketsAgent",
      },
    },
    {
      label: __("New Ticket"),
      route: {
        name: "TicketNew",
      },
    },
  ];
  return items;
});

usePageMeta(() => ({
  title: __("New Ticket"),
}));

onMounted(() => {
  capture("new_ticket_page", {
    data: {
      user: userID,
    },
  });
});
</script>
