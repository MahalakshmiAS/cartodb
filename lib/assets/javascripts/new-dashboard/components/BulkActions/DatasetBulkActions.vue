<template>
  <BulkActions :actions="actions[actionMode]" v-on="getEventListeners()"></BulkActions>
</template>

<script>
import BulkActions from 'new-dashboard/components/BulkActions/BulkActions';
import * as DialogActions from 'new-dashboard/core/dialog-actions';
import * as Table from 'new-dashboard/core/models/table';
import * as Visualization from 'new-dashboard/core/models/visualization';

export default {
  name: 'DatasetBulkActions',
  inject: ['backboneViews'],
  components: {
    BulkActions
  },
  props: {
    areAllDatasetsSelected: {
      type: Boolean,
      default: false
    },
    selectedDatasets: {
      type: Array,
      required: true
    }
  },
  computed: {
    actions () {
      return {
        single: [
          {
            name: this.$t('BulkActions.datasets.selectAllDatasets'),
            event: 'selectAll',
            shouldBeHidden: this.areAllMapsSelected
          },
          {
            name: this.$t('BulkActions.datasets.createMap'),
            event: 'createMap',
            shouldBeHidden: this.isAnyLocked
          },
          {
            name: this.$t('BulkActions.datasets.changeDatasetPrivacy'),
            event: 'changePrivacy',
            shouldBeHidden: this.isAnyShared || this.isAnyLocked
          },
          {
            name: this.$t('BulkActions.datasets.duplicateDataset'),
            event: 'duplicateDataset'
          },
          {
            name: this.$t('BulkActions.datasets.lockDataset'),
            event: 'lockDataset',
            shouldBeDisabled: this.isAnyShared && !this.areAllLocked,
            shouldBeHidden: this.isAnyLocked
          },
          {
            name: this.$t('BulkActions.datasets.unlockDataset'),
            event: 'unlockDataset',
            shouldBeDisabled: this.isAnyShared && this.areAllLocked,
            shouldBeHidden: !this.areAllLocked
          },
          {
            name: this.$t('BulkActions.datasets.deleteDataset'),
            event: 'deleteDataset',
            isDestructive: true,
            shouldBeDisabled: this.isAnyShared && !this.isAnyLocked,
            shouldBeHidden: this.isAnyLocked
          }
        ],
        multiple: [
          {
            name: this.$t('BulkActions.datasets.selectAllDatasets'),
            event: 'selectAll',
            shouldBeHidden: this.areAllDatasetsSelected
          },
          {
            name: this.$t('BulkActions.datasets.deselectAllDatasets'),
            event: 'deselectAll'
          },
          {
            name: this.$t('BulkActions.datasets.createMap'),
            event: 'createMap',
            shouldBeHidden: this.isAnyLocked
          },
          {
            name: this.$t('BulkActions.datasets.lockDatasets'),
            event: 'lockDatasets',
            shouldBeDisabled: this.isAnyShared && !this.areAllLocked,
            shouldBeHidden: this.isAnyLocked
          },
          {
            name: this.$t('BulkActions.datasets.unlockDatasets'),
            event: 'unlockDatasets',
            shouldBeDisabled: this.isAnyShared && this.areAllLocked,
            shouldBeHidden: !this.areAllLocked
          },
          {
            name: this.$t('BulkActions.datasets.deleteDatasets'),
            event: 'deleteDatasets',
            isDestructive: true,
            shouldBeDisabled: this.isAnyShared && !this.isAnyLocked,
            shouldBeHidden: this.isAnyLocked
          }
        ]
      };
    },
    actionMode () {
      return this.selectedDatasets.length > 1 ? 'multiple' : 'single';
    },
    isAnyShared () {
      return this.selectedDatasets.some(dataset => Visualization.isSharedWithMe(dataset, this.$cartoModels));
    },
    isAnyLocked () {
      return this.selectedDatasets.some(dataset => dataset.locked);
    },
    areAllLocked () {
      return this.selectedDatasets.every(dataset => dataset.locked);
    }
  },
  methods: {
    getActionHandlers () {
      return {
        deselectAll: () => {
          this.deselectAll();
        },
        fetchList: () => {
          this.$store.dispatch('datasets/fetch');
        },
        updateVisualization: (model) => {
          this.$store.dispatch('datasets/updateVisualization', { visualizationId: model.get('id'), visualizationAttributes: model.attributes });
        }
      };
    },
    getEventListeners () {
      const events = this.actions[this.actionMode].map(action => action.event);

      return events.reduce(
        (eventListeners, action) => {
          eventListeners[action] = this[action].bind(this);
          return eventListeners;
        }, {}
      );
    },
    showModal (componentDefinition, componentPropsData) {
      this.$modal.show(
        componentDefinition,
        componentPropsData,
        { width: '100%', height: '100%' }
      );
    },
    selectAll () {
      this.$emit('selectAll');
    },
    deselectAll () {
      this.$emit('deselectAll');
    },
    createMap () {
      DialogActions.createMap.apply(this, [
        this.selectedDatasets,
        this.backboneViews.backgroundPollingView.getBackgroundPollingView(),
        this.backboneViews.mamufasImportView.getView()
      ]);
    },
    changePrivacy () {
      DialogActions.changePrivacy.apply(this, [this.selectedDatasets[0], this.getActionHandlers()]);
    },
    duplicateDataset () {
      const selectedDataset = this.selectedDatasets[0];
      const bgPollingView = this.backboneViews.backgroundPollingView.getBackgroundPollingView();

      bgPollingView._addDataset({
        type: 'duplication',
        table_name: `${Table.getUnqualifiedName(selectedDataset.name)}_copy`,
        value: `${selectedDataset.permission.owner.username}.${selectedDataset.name}`,
        create_vis: false
      });
      this.deselectAll();
    },
    unlockDataset () {
      DialogActions.changeLockState.apply(this, [
        this.selectedDatasets[0],
        'datasets',
        this.getActionHandlers()
      ]);
    },
    lockDataset () {
      DialogActions.changeLockState.apply(this, [
        this.selectedDatasets[0],
        'datasets',
        this.getActionHandlers()
      ]);
    },
    deleteDataset () {
      DialogActions.deleteVisualization.apply(this, [
        this.selectedDatasets[0],
        'datasets',
        this.getActionHandlers()
      ]);
    },
    unlockDatasets () {
      DialogActions.changeVisualizationsLockState.apply(this, [
        this.selectedDatasets,
        'datasets',
        this.getActionHandlers()
      ]);
    },
    lockDatasets () {
      DialogActions.changeVisualizationsLockState.apply(this, [
        this.selectedDatasets,
        'datasets',
        this.getActionHandlers()
      ]);
    },
    deleteDatasets () {
      DialogActions.deleteVisualizations.apply(this, [
        this.selectedDatasets,
        'datasets',
        this.getActionHandlers()
      ]);
    }
  }
};
</script>
