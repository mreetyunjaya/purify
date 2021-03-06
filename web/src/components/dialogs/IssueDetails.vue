<template>
  <v-card class="fill-height">
    <v-card-title>
      <v-container>
        <v-row>
          <v-col cols="1" class="ml-2">
            <v-row justify="center">
              <v-tooltip bottom>
                <template v-slot:activator="{ on }">
                  <v-btn
                    icon
                    large
                    :color="genColor()"
                    @click="riskDialog = !riskDialog"
                    v-on="on"
                  >
                    <v-icon>fa-bug</v-icon>
                  </v-btn>
                </template>
                <span>Change Risk</span>
              </v-tooltip>
            </v-row>
            <v-row justify="center">
              <span class="text-capitalize">{{ issue.risk }}</span>
            </v-row>
            <v-dialog v-model="riskDialog" max-width="300">
              <v-card>
                <v-card-title>
                  <span class="title mb-2">Change risk</span>
                </v-card-title>
                <v-card-text>
                  <v-select
                    v-model="risk"
                    :items="severities"
                    label="How bad is it?"
                    outlined
                    dense
                  />
                </v-card-text>
                <v-divider />
                <v-card-actions>
                  <v-spacer />
                  <v-btn
                    color="green darken-1"
                    text
                    @click="updateIssue(issue, 'risk', risk)"
                  >
                    Confirm
                  </v-btn>
                </v-card-actions>
              </v-card>
            </v-dialog>
          </v-col>
          <v-divider class="my-3 mx-3" vertical />
          <v-col class="ml-2" cols="9">
            <v-row class="headline">
              {{ issue.title }}
            </v-row>
            <v-row
              class="title grey--text font-weight-light my-2"
            >
              {{ issue.subtitle }}
            </v-row>
          </v-col>
        </v-row>
        <v-row justify="center">
          <v-col>
            <v-menu
              v-if="issue.status === 'open'"
              transition="slide-y-transition"
              bottom
            >
              <template v-slot:activator="{ on }">
                <v-btn
                  class="mr-2"
                  color="tertiary"
                  outlined
                  v-on="on"
                >
                  Resolution
                  <v-icon right>
                    mdi-chevron-down
                  </v-icon>
                </v-btn>
              </template>
              <v-list>
                <!-- eslint-disable-next-line max-len -->
                <v-list-item
                  key="switch-resolution"
                  @click="updateIssue(issue, 'resolution', 'resolved')"
                >
                  <v-list-item-title>Resolved</v-list-item-title>
                </v-list-item>
                <v-list-item
                  key="switch-as-fp"
                  @click="updateIssue(issue, 'resolution', 'false positive')"
                >
                  <v-list-item-title>False Positive</v-list-item-title>
                </v-list-item>
                <!-- eslint-disable-next-line max-len -->
                <v-list-item
                  key="switch-risk-accepted"
                  @click="updateIssue(issue, 'resolution', 'accepted risk')"
                >
                  <v-list-item-title>Accepted Risk</v-list-item-title>
                </v-list-item>
              </v-list>
            </v-menu>
            <v-btn
              v-else
              outlined
              class="mr-2"
              color="primary"
              @click="updateIssue(issue, 'status', 'open')"
            >
              Reopen
            </v-btn>
            <v-tooltip bottom>
              <template v-slot:activator="{ on }">
                <span v-on="on">
                  <v-btn
                    v-if="!issue.ticket"
                    outlined
                    :disabled="!systemSetup.jira"
                    class="mr-2"
                    color="senary"
                    @click="ticketDialog = !ticketDialog"
                  >
                    <v-icon color="senary" left>mdi-jira</v-icon>Create Ticket
                  </v-btn>
                </span>
              </template>
              <span>Set up Jira to create a ticket</span>
            </v-tooltip>
            <v-btn
              v-if="issue.ticket"
              outlined
              class="mx-2"
              color="senary"
              :href="issue.ticket.link"
              target="_blank"
            >
              <v-icon
                v-if="issue.ticket.type == 'jira'"
                color="senary"
                left
              >
                mdi-jira
              </v-icon>
              {{ issue.ticket.key }}
            </v-btn>
            <v-btn
              outlined
              class="mr-2"
              color="quaternary"
              @click="editDialog = true"
            >
              <v-icon left>
                mdi-pencil
              </v-icon>Edit
            </v-btn>
            <v-btn
              outlined
              class="mr-2"
              color="secondary"
              @click="commentDialog = true"
            >
              <v-icon :left="!!issueComments.length" small>
                mdi-comment-text-multiple
              </v-icon>
              <span v-if="issueComments.length">{{ issueComments.length }}</span>
            </v-btn>
            <v-dialog
              :key="`edit-dialog-${issue._id}`"
              v-model="editDialog"
              max-width="800"
            >
              <edit-issue-dialog :issue.sync="issue" />
            </v-dialog>
            <jira-ticket-dialog
              :key="`ticket-dialog-${issue._id}`"
              :issue.sync="issue"
              :dialog.sync="ticketDialog"
            />
            <comment-dialog
              :key="`idcd-${issue._id}`"
              :issue-id.sync="issue._id"
              :dialog.sync="commentDialog"
            />
          </v-col>
        </v-row>
      </v-container>
    </v-card-title>
    <v-container>
      <v-col v-for="field in issueTemplate.body_fields" :key="`id-${field.key}`">
        <fields-parser :ikey="field" :ivalue="getValue(issue.fields, field.key)" />
      </v-col>
    </v-container>
  </v-card>
</template>
<script>
/* eslint-disable no-restricted-syntax */
import { mapState, mapGetters } from 'vuex';
import FieldsParser from '@/components/FieldsParser.vue';
import JiraTicketDialog from '@/components/dialogs/JiraTicketDialog.vue';
import EditIssueDialog from '@/components/dialogs/EditIssueDialog.vue';
import CommentDialog from '@/components/dialogs/CommentDialog.vue';
import { ISSUE_UPDATE } from '@/store/actions';
import { matchPattern, parseKey, getValue } from '@//utils/helpers';

export default {
  name: 'IssueDetails',

  components: {
    JiraTicketDialog,
    FieldsParser,
    CommentDialog,
    EditIssueDialog,
  },

  props: {
    issue: {
      type: Object,
      required: true,
      default: () => {},
    },
  },

  data() {
    return {
      ticketDialog: false,
      editDialog: false,
      severities: ['info', 'low', 'medium', 'high', 'critical'],
      riskDialog: false,
      commentDialog: false,
      risk: '',
    };
  },

  computed: {
    ...mapState({
      systemSetup: (state) => state.app.setup,
      issueComments: (state) => state.issues.comments,
    }),

    ...mapGetters(['findTemplateByName']),

    preparedMarkdown() {
      let result = '';
      for (const field of this.issueTemplate.body_fields) {
        result += `## ${this.parseKey(field.key)}\n`;
        result += `${this.getValue(this.issue.fields, field.key)}\n\n`;
      }
      return result;
    },

    issueTemplate() {
      return this.findTemplateByName(this.issue.template).template;
    },
  },

  methods: {
    matchPattern,

    parseKey,

    getValue,

    isPrintable(obj) {
      return ['string', 'boolean', 'number'].includes(typeof obj);
    },

    updateIssue(item, field, value) {
      const change = {};
      if (field === 'resolution') {
        change.status = 'closed';
      } else if (field === 'status') {
        change.resolution = 'none';
      }

      change[field] = value;

      this.$store.dispatch(ISSUE_UPDATE, { ids: [item._id], change }).then(() => {
        if (field === 'resolution') {
          this.issue.status = 'closed';
        } else if (field === 'status') {
          this.resolution = 'none';
        }
        this.issue[field] = value;

        if (field === 'risk') {
          this.riskDialog = false;
        }

        this.$showSuccessMessage('The issue has been updated');
      });
    },

    genColor() {
      switch (this.issue.risk) {
        case 'info':
          return 'light-blue lighten-3';
        case 'low':
          return 'blue';
        case 'medium':
          return 'orange';
        case 'high':
          return 'red';
        case 'critical':
          return 'red darken-4';
        default:
          return 'grey';
      }
    },
  },
};
</script>
