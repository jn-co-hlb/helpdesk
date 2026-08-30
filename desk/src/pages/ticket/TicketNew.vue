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
        <!-- HLB-FORK: request-spec — what to write, directly above where they
             write it. See the comment on requestSpec below.

             Not in a bordered box: the type notice above is already a box, and
             two stacked boxes read as chrome rather than as the one thing on
             the form somebody actually has to act on. Colour carries it
             instead, and the extra top margin keeps it off the field above. -->
        <div v-if="requestSpec.length" class="mt-0.5 flex flex-col gap-2">
          <span class="block text-sm text-ink-gray-7">
            {{ __("Request Specification") }}
          </span>
          <div class="text-p-sm text-ink-blue-5">
            <p>
              {{
                __(
                  "The following information should be provided below without which the request cannot be initiated."
                )
              }}
            </p>
            <ol
              v-if="requestSpec.length > 1"
              class="mt-1.5 list-decimal ps-5 italic"
            >
              <li v-for="(item, index) in requestSpec" :key="index">
                {{ item }}
              </li>
            </ol>
            <!-- A single requirement is a sentence, not a list of one. -->
            <p v-else class="mt-1.5 italic">{{ requestSpec[0] }}</p>
          </div>
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

// HLB-FORK: editor-hint — the same idea one level up. Operational Incident,
// Security Incident and Project & Innovation want ONE prompt for the whole type
// rather than a different one per sub-type, and each has just retired several
// narrative fields into the body, so the body has to ask for what they held.
const TYPE_EDITOR_HINTS: Record<string, string> = {
  "Operational Incident":
    "Please describe the issue in detail, including what you were busy with at the time and what you saw on screen. Paste or attach screenshots and any supporting documentation.",
  "Security Incident":
    "Please supply as much detail as possible, including but not limited to what preceded the incident and what has already been done in response to it.",
  "Project & Innovation Request":
    "Please describe the problem or opportunity, how this is done today, the outcome you want and how you would measure success, the systems and data involved, the expected benefit, and any indicative budget.",
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
  // Sub-type first: it is the more specific answer, so a type-wide prompt never
  // masks one written for a particular sub-type.
  for (const fieldname of SUBTYPE_FIELDS) {
    const hint = EDITOR_HINTS[fields[fieldname]];
    if (hint) return __(hint);
  }
  const typeHint = TYPE_EDITOR_HINTS[fields["ticket_type"]];
  if (typeHint) return __(typeHint);
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
    "GITC & IT Audit Request": {
      label: "Client Name",
      placeholder: "The client this audit work is for.",
    },
  };

// HLB-FORK: gitc-year-end — stamp the year end onto the subject on submit.
// An audit queue is read by client AND period: "Acme Ltd" alone is ambiguous
// the moment the next year's work starts, and asking people to type the
// convention by hand produces four spellings of it. Composed at submit rather
// than while typing so the field stays a plain client name to fill in.
const GITC_TICKET_TYPE = "GITC & IT Audit Request";
const SUBJECT_MAX = 140;

function composeSubject(): string {
  const fields = templateFields as Record<string, string>;
  const base = subject.value.trim();
  if (fields["ticket_type"] !== GITC_TICKET_TYPE) return base;
  const yearEnd = (fields["au_year_end"] || "").trim();
  if (!base || !yearEnd) return base;
  const suffix = `YE: ${yearEnd}`;
  // Idempotent: a resubmit after a validation error must not stack suffixes.
  if (base.endsWith(suffix)) return base;
  // HD Ticket.subject is a Data field — varchar(140) — and the input above
  // already lets someone use all 140. Appending would push it over and frappe
  // aborts the insert with CharacterLengthExceededError, so trim the client
  // name rather than lose the period.
  const room = SUBJECT_MAX - suffix.length - 1;
  const head = base.length > room ? base.slice(0, room).trimEnd() : base;
  return `${head} ${suffix}`;
}

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
  // HLB-FORK: request-spec — hlb_spec is our Custom Field on HD Ticket Type.
  fields: ["name", "description", "hlb_spec"],
  auto: true,
  cache: "ticketTypes",
});

const ticketTypeNotice = computed(() => {
  const selected = (templateFields as Record<string, string>)["ticket_type"];
  if (!selected) return "";
  return ticketTypeResource.dataMap?.[selected]?.description?.trim() || "";
});

// HLB-FORK: request-spec — the numbered list of things a submitter has to put
// in the body before the request can be worked.
//
// The type notice above says which BOX to tick; this says what to WRITE, and
// they are different jobs done at different moments, so it is a second block
// rather than more words in the first one. It sits directly above the editor
// because that is where it has to be readable while typing — a placeholder
// disappears at the first keystroke, which is exactly when the list is needed.
//
// The content lives on the HD Ticket Type record (Custom Field `hlb_spec`, one
// requirement per line), not in a map in this file: the lists differ per type
// and will be rewritten as the SOP settles, and none of that should need a
// fork commit, a tag and a rebuild.
// See customisations.manifest.json id=ui-request-spec.
// HLB-FORK: request-spec — sub-types whose specification differs from their
// ticket type's. Keyed on the sub-type VALUE like EDITOR_HINTS above, because
// those values are already unique across the whole form.
//
// These live here and not on a record because a sub-type IS a string in a
// Select — there is no per-sub-type document to hang a field on. The type-level
// default still comes from HD Ticket Type.hlb_spec, so only the exceptions cost
// a fork commit.
const SUBTYPE_SPECS: Record<string, string> = {
  "Design and Implementation":
    "Please enter all the information related to the request as well as all client details and all systems in scope, in the below text-area.",
  "Operating Effectiveness":
    "Please enter all the information related to the request as well as all client details and all systems in scope, in the below text-area.",
};

const requestSpec = computed<string[]>(() => {
  const fields = templateFields as Record<string, string>;
  const selected = fields["ticket_type"];
  if (!selected) return [];
  // Sub-type first: it is the more specific answer, so a type-wide spec never
  // masks one written for a particular sub-type.
  const override = SUBTYPE_FIELDS.map((f) => SUBTYPE_SPECS[fields[f]]).find(
    Boolean
  );
  const raw = override || ticketTypeResource.dataMap?.[selected]?.hlb_spec || "";
  return raw
    .split("\n")
    .map((line: string) => line.trim())
    .filter((line: string) => line.length > 0);
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

// HLB-FORK: customer-priority — ticket types where the SUBMITTER picks the
// priority. Priority is hidden from the customer portal everywhere else, via
// HLB-FORK: customer-priority RETIRED 2026-08-30. Priority is no longer on the
// intake template at all — no ticket type asks for it at creation, agents set
// it on the ticket afterwards — so the per-type exception this carried has
// nothing left to except. Removed rather than left inert: an unused branch in
// a forked file is one more thing to re-land on the next rebase for no
// behaviour. See customisations.manifest.json id=ui-customer-priority.
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
      // HLB-FORK: gitc-year-end
      subject: composeSubject(),
      template: props.templateId,
      ...templateFields,
    },
    attachments: attachments.value,
  }),
  // HLB-FORK: validate-visible — only demand what the form actually drew.
  //
  // `visibleFields` is the whole template, not the visible part of it: the
  // hiding happens one level down, in UniInput's own
  // `v-if="field.display_via_depends_on"`. Filtering on `required` alone
  // therefore demands fields that are not on screen, and names them in an
  // error the person has no way to act on. On 2026-08-25 a single stray
  // `required: 1` on a Project & Innovation template row blocked submission of
  // EVERY ticket type this way — "Fieldwork completion date is required" on a
  // printer request. The config half is fixed in company_helpdesk; this is the
  // half that made one bad row an outage instead of a stray asterisk.
  //
  // `description` had the same shape of bug: it was unconditional here while
  // the Submit button already honoured `bodyRequired`, so an Offboarding
  // ticket — where optional-body deliberately allows an empty body — offered
  // an enabled button and then refused the submit.
  // See customisations.manifest.json id=ui-validate-visible.
  validate: (params) => {
    const fields =
      visibleFields.value?.filter(
        (f) => f.required && f.display_via_depends_on
      ) || [];
    const toVerify: any[] = [...fields, "subject"];
    if (bodyRequired.value) toVerify.push("description");
    for (const field of toVerify) {
      if (!params.doc[field.fieldname || field]) {
        const label = field.label ? __(field.label) : field;
        return `${label} is required`;
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
