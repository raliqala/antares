<template>
   <div
      v-show="isSelected"
      class="workspace-query-tab column col-12 columns col-gapless no-outline p-0"
      tabindex="0"
      @keydown.116="runQuery(query)"
      @keydown.75="killTabQuery"
      @keydown.ctrl.alt.87="clear"
      @keydown.ctrl.66="beautify"
      @keydown.ctrl.71="openHistoryModal"
   >
      <div class="workspace-query-runner column col-12">
         <QueryEditor
            v-show="isSelected"
            ref="queryEditor"
            :auto-focus="true"
            :value.sync="query"
            :workspace="workspace"
            :schema="breadcrumbsSchema"
            :is-selected="isSelected"
            :height="editorHeight"
         />
         <div ref="resizer" class="query-area-resizer" />
         <div class="workspace-query-runner-footer">
            <div class="workspace-query-buttons">
               <div @mouseenter="setCancelButtonVisibility(true)" @mouseleave="setCancelButtonVisibility(false)">
                  <button
                     v-if="showCancel && isQuering"
                     class="btn btn-primary btn-sm cancellable"
                     :disabled="!query"
                     :title="$t('word.cancel')"
                     @click="killTabQuery()"
                  >
                     <i class="mdi mdi-24px mdi-window-close" />
                     <span class="d-invisible pr-1">{{ $t('word.run') }}</span>
                  </button>
                  <button
                     v-else
                     class="btn btn-primary btn-sm"
                     :class="{'loading':isQuering}"
                     :disabled="!query"
                     title="F5"
                     @click="runQuery(query)"
                  >
                     <i class="mdi mdi-24px mdi-play pr-1" />
                     <span>{{ $t('word.run') }}</span>
                  </button>
               </div>
               <button
                  v-if="!autocommit"
                  class="btn btn-dark btn-sm"
                  :class="{'loading':isQuering}"
                  @click="commitTab()"
               >
                  <i class="mdi mdi-24px mdi-cube-send pr-1" />
                  <span>{{ $t('word.commit') }}</span>
               </button>
               <button
                  v-if="!autocommit"
                  class="btn btn-dark btn-sm"
                  :class="{'loading':isQuering}"
                  @click="rollbackTab()"
               >
                  <i class="mdi mdi-24px mdi-undo-variant pr-1" />
                  <span>{{ $t('word.rollback') }}</span>
               </button>
               <button
                  class="btn btn-link btn-sm mr-0"
                  :disabled="!query || isQuering"
                  title="CTRL+W"
                  @click="clear()"
               >
                  <i class="mdi mdi-24px mdi-delete-sweep pr-1" />
                  <span>{{ $t('word.clear') }}</span>
               </button>

               <div class="divider-vert py-3" />

               <button
                  class="btn btn-dark btn-sm"
                  :disabled="!query || isQuering"
                  title="CTRL+B"
                  @click="beautify()"
               >
                  <i class="mdi mdi-24px mdi-brush pr-1" />
                  <span>{{ $t('word.format') }}</span>
               </button>
               <button
                  class="btn btn-dark btn-sm"
                  :disabled="isQuering"
                  title="CTRL+G"
                  @click="openHistoryModal()"
               >
                  <i class="mdi mdi-24px mdi-history pr-1" />
                  <span>{{ $t('word.history') }}</span>
               </button>
               <div class="dropdown table-dropdown pr-2">
                  <button
                     :disabled="!hasResults || isQuering"
                     class="btn btn-dark btn-sm dropdown-toggle mr-0 pr-0"
                     tabindex="0"
                  >
                     <i class="mdi mdi-24px mdi-file-export mr-1" />
                     <span>{{ $t('word.export') }}</span>
                     <i class="mdi mdi-24px mdi-menu-down" />
                  </button>
                  <ul class="menu text-left">
                     <li class="menu-item">
                        <a class="c-hand" @click="downloadTable('json')">JSON</a>
                     </li>
                     <li class="menu-item">
                        <a class="c-hand" @click="downloadTable('csv')">CSV</a>
                     </li>
                  </ul>
               </div>
               <div class="input-group pr-2" :title="$t('message.commitMode')">
                  <i class="input-group-addon addon-sm mdi mdi-24px mdi-source-commit p-0" />
                  <select v-model="autocommit" class="form-select select-sm text-bold">
                     <option :value="true">
                        {{ $t('message.autoCommit') }}
                     </option>
                     <option :value="false">
                        {{ $t('message.manualCommit') }}
                     </option>
                  </select>
               </div>
            </div>
            <div class="workspace-query-info">
               <div
                  v-if="results.length"
                  class="d-flex"
                  :title="$t('message.queryDuration')"
               >
                  <i class="mdi mdi-timer-sand mdi-rotate-180 pr-1" /> <b>{{ durationsCount / 1000 }}s</b>
               </div>
               <div
                  v-if="resultsCount"
                  class="d-flex"
                  :title="$t('word.results')"
               >
                  <i class="mdi mdi-equal pr-1" /> <b>{{ resultsCount.toLocaleString() }}</b>
               </div>
               <div
                  v-if="hasAffected"
                  class="d-flex"
                  :title="$t('message.affectedRows')"
               >
                  <i class="mdi mdi-target pr-1" /> <b>{{ affectedCount }}</b>
               </div>
               <div class="input-group" :title="$t('word.schema')">
                  <i class="input-group-addon addon-sm mdi mdi-24px mdi-database" />
                  <select v-model="selectedSchema" class="form-select select-sm text-bold">
                     <option :value="null">
                        {{ $t('message.noSchema') }}
                     </option>
                     <option v-for="schemaName in databaseSchemas" :key="schemaName">
                        {{ schemaName }}
                     </option>
                  </select>
               </div>
            </div>
         </div>
      </div>
      <WorkspaceTabQueryEmptyState v-if="!results.length && !isQuering" :customizations="workspace.customizations" />
      <div class="workspace-query-results p-relative column col-12">
         <BaseLoader v-if="isQuering" />
         <WorkspaceTabQueryTable
            v-if="results"
            v-show="!isQuering"
            ref="queryTable"
            :results="results"
            :tab-uid="tab.uid"
            :conn-uid="connection.uid"
            :is-selected="isSelected"
            mode="query"
            @update-field="updateField"
            @delete-selected="deleteSelected"
         />
      </div>
      <ModalHistory
         v-if="isHistoryOpen"
         :connection="connection"
         @select-query="selectQuery"
         @close="isHistoryOpen = false"
      />
   </div>
</template>

<script>
import { format } from 'sql-formatter';
import { mapGetters, mapActions } from 'vuex';
import Schema from '@/ipc-api/Schema';
import QueryEditor from '@/components/QueryEditor';
import BaseLoader from '@/components/BaseLoader';
import WorkspaceTabQueryTable from '@/components/WorkspaceTabQueryTable';
import WorkspaceTabQueryEmptyState from '@/components/WorkspaceTabQueryEmptyState';
import ModalHistory from '@/components/ModalHistory';
import tableTabs from '@/mixins/tableTabs';

export default {
   name: 'WorkspaceTabQuery',
   components: {
      BaseLoader,
      QueryEditor,
      WorkspaceTabQueryTable,
      WorkspaceTabQueryEmptyState,
      ModalHistory
   },
   mixins: [tableTabs],
   props: {
      connection: Object,
      tab: Object,
      isSelected: Boolean
   },
   data () {
      return {
         query: '',
         lastQuery: '',
         isQuering: false,
         isCancelling: false,
         showCancel: false,
         autocommit: true,
         results: [],
         selectedSchema: null,
         resultsCount: 0,
         durationsCount: 0,
         affectedCount: null,
         editorHeight: 200,
         isHistoryOpen: false
      };
   },
   computed: {
      ...mapGetters({
         getWorkspace: 'workspaces/getWorkspace',
         selectedWorkspace: 'workspaces/getSelected',
         getHistoryByWorkspace: 'history/getHistoryByWorkspace'
      }),
      workspace () {
         return this.getWorkspace(this.connection.uid);
      },
      tabUid () {
         return this.$vnode.key;
      },
      breadcrumbsSchema () {
         return this.workspace.breadcrumbs.schema || null;
      },
      databaseSchemas () {
         return this.workspace.structure.reduce((acc, curr) => {
            acc.push(curr.name);
            return acc;
         }, []);
      },
      isWorkspaceSelected () {
         return this.workspace.uid === this.selectedWorkspace;
      },
      history () {
         return this.getHistoryByWorkspace(this.connection.uid) || [];
      },
      hasResults () {
         return this.results.length && this.results[0].rows;
      },
      hasAffected () {
         return this.affectedCount || (!this.resultsCount && this.affectedCount !== null);
      }
   },
   watch: {
      isSelected (val) {
         if (val) {
            this.changeBreadcrumbs({ schema: this.selectedSchema, query: `Query #${this.tab.index}` });
            setTimeout(() => {
               if (this.$refs.queryEditor)
                  this.$refs.queryEditor.editor.focus();
            }, 0);
         }
      },
      selectedSchema () {
         this.changeBreadcrumbs({ schema: this.selectedSchema, query: `Query #${this.tab.index}` });
      }
   },
   created () {
      this.query = this.tab.content;
      this.selectedSchema = this.tab.schema || this.breadcrumbsSchema;

      window.addEventListener('keydown', this.onKey);
   },
   mounted () {
      const resizer = this.$refs.resizer;

      resizer.addEventListener('mousedown', e => {
         e.preventDefault();

         window.addEventListener('mousemove', this.resize);
         window.addEventListener('mouseup', this.stopResize);
      });

      if (this.tab.autorun)
         this.runQuery(this.query);
   },
   beforeDestroy () {
      window.removeEventListener('keydown', this.onKey);
      const params = {
         uid: this.connection.uid,
         tabUid: this.tab.uid
      };
      Schema.destroyConnectionToCommit(params);
   },
   methods: {
      ...mapActions({
         addNotification: 'notifications/addNotification',
         changeBreadcrumbs: 'workspaces/changeBreadcrumbs',
         updateTabContent: 'workspaces/updateTabContent',
         setUnsavedChanges: 'workspaces/setUnsavedChanges',
         saveHistory: 'history/saveHistory'
      }),
      async runQuery (query) {
         if (!query || this.isQuering) return;
         this.isQuering = true;
         this.clearTabData();
         this.$refs.queryTable.resetSort();

         try { // Query Data
            const params = {
               uid: this.connection.uid,
               schema: this.selectedSchema,
               tabUid: this.tab.uid,
               autocommit: this.autocommit,
               query
            };

            const { status, response } = await Schema.rawQuery(params);

            if (status === 'success') {
               this.results = Array.isArray(response) ? response : [response];
               this.resultsCount = this.results.reduce((acc, curr) => acc + (curr.rows ? curr.rows.length : 0), 0);
               this.durationsCount = this.results.reduce((acc, curr) => acc + curr.duration, 0);
               this.affectedCount = this.results
                  .filter(result => result.report !== null)
                  .reduce((acc, curr) => {
                     if (acc === null) acc = 0;
                     return acc + (curr.report ? curr.report.affectedRows : 0);
                  }, null);

               this.updateTabContent({
                  uid: this.connection.uid,
                  tab: this.tab.uid,
                  type: 'query',
                  schema: this.selectedSchema,
                  content: query
               });
               this.saveHistory(params);
               if (!this.autocommit)
                  this.setUnsavedChanges({ uid: this.connection.uid, tUid: this.tabUid, isChanged: true });
            }
            else
               this.addNotification({ status: 'error', message: response });
         }
         catch (err) {
            this.addNotification({ status: 'error', message: err.stack });
         }

         this.isQuering = false;
         this.lastQuery = query;
      },
      async killTabQuery () {
         if (this.isCancelling) return;

         this.isCancelling = true;

         try {
            const params = {
               uid: this.connection.uid,
               tabUid: this.tab.uid
            };

            await Schema.killTabQuery(params);
         }
         catch (err) {
            this.addNotification({ status: 'error', message: err.stack });
         }

         this.isCancelling = false;
      },
      setCancelButtonVisibility (val) {
         if (this.workspace.customizations.cancelQueries)
            this.showCancel = val;
      },
      reloadTable () {
         this.runQuery(this.lastQuery);
      },
      clearTabData () {
         this.results = [];
         this.resultsCount = 0;
         this.durationsCount = 0;
         this.affectedCount = null;
      },
      resize (e) {
         const el = this.$refs.queryEditor.$el;
         let editorHeight = e.pageY - el.getBoundingClientRect().top;
         if (editorHeight > 400) editorHeight = 400;
         if (editorHeight < 50) editorHeight = 50;
         this.editorHeight = editorHeight;
      },
      stopResize () {
         window.removeEventListener('mousemove', this.resize);
         if (this.$refs.queryTable && this.results.length)
            this.$refs.queryTable.resizeResults();

         if (this.$refs.queryEditor)
            this.$refs.queryEditor.editor.resize();
      },
      beautify () {
         if (this.$refs.queryEditor) {
            let language = 'sql';

            switch (this.workspace.client) {
               case 'mysql':
                  language = 'mysql';
                  break;
               case 'maria':
                  language = 'mariadb';
                  break;
               case 'pg':
                  language = 'postgresql';
                  break;
            }

            const formattedQuery = format(this.query, {
               language,
               uppercase: true
            });
            this.$refs.queryEditor.editor.session.setValue(formattedQuery);
         }
      },
      openHistoryModal () {
         this.isHistoryOpen = true;
      },
      selectQuery (sql) {
         if (this.$refs.queryEditor)
            this.$refs.queryEditor.editor.session.setValue(sql);

         this.isHistoryOpen = false;
      },
      clear () {
         if (this.$refs.queryEditor)
            this.$refs.queryEditor.editor.session.setValue('');
         this.clearTabData();
      },
      downloadTable (format) {
         this.$refs.queryTable.downloadTable(format, `${this.tab.type}-${this.tab.index}`);
      },
      async commitTab () {
         this.isQuering = true;
         try {
            const params = {
               uid: this.connection.uid,
               tabUid: this.tab.uid
            };

            await Schema.commitTab(params);
            this.setUnsavedChanges({ uid: this.connection.uid, tUid: this.tabUid, isChanged: false });
            this.addNotification({ status: 'success', message: this.$t('message.actionSuccessful', { action: 'COMMIT' }) });
         }
         catch (err) {
            this.addNotification({ status: 'error', message: err.stack });
         }

         this.isQuering = false;
      },
      async rollbackTab () {
         this.isQuering = true;
         try {
            const params = {
               uid: this.connection.uid,
               tabUid: this.tab.uid
            };

            await Schema.rollbackTab(params);
            this.setUnsavedChanges({ uid: this.connection.uid, tUid: this.tabUid, isChanged: false });
            this.addNotification({ status: 'success', message: this.$t('message.actionSuccessful', { action: 'ROLLBACK' }) });
         }
         catch (err) {
            this.addNotification({ status: 'error', message: err.stack });
         }

         this.isQuering = false;
      }
   }
};
</script>

<style lang="scss">
.workspace-tabs {
  align-content: baseline;

  .workspace-query-runner {
    position: relative;

    .query-area-resizer {
      position: absolute;
      height: 4px;
      bottom: 40px;
      width: 100%;
      cursor: ns-resize;
      z-index: 99;
      transition: background 0.2s;

      &:hover {
        background: rgba($primary-color, 50%);
      }
    }

    .workspace-query-runner-footer {
      display: flex;
      flex-wrap: wrap;
      row-gap: 0.4rem;
      justify-content: space-between;
      padding: 0.3rem 0.6rem 0.4rem;
      align-items: center;
      min-height: 42px;

      .workspace-query-buttons,
      .workspace-query-info {
        display: flex;
        align-items: center;

        .btn {
          display: flex;
          align-self: center;
          margin-right: 0.4rem;
        }
      }

      .workspace-query-info {
        overflow: visible;

        > div + div {
          padding-left: 0.6rem;
        }
      }
    }
  }

  .workspace-query-results {
    min-height: 200px;
  }
}
</style>
