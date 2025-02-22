<template>
  <div :class="{'subplan': node[nodeProps.SUBPLAN_NAME], 'd-flex flex-column align-items-center': viewOptions.orientation == orientations.TWOD}">
    <h4 v-if="node[nodeProps.SUBPLAN_NAME]">{{ node[nodeProps.SUBPLAN_NAME] }}</h4>
    <div
      :class="['text-left plan-node', {
        'detailed': showDetails,
        'never-executed': isNeverExecuted,
        'parallel': workersPlannedCount,
        'selected': selected
      }]"
    >
      <div class="workers text-muted py-0 px-1" v-if="workersPlannedCount">
        <div v-for="index in workersPlannedCountReversed" :style="'top: ' + (1 + index * 2)  + 'px; left: ' + (1 + (index + 1) * 3) + 'px;'"
             :class="{'border-dashed': index >= workersLaunchedCount}">
          {{ index }}
        </div>
      </div>
      <div class="collapse-handle" v-if="hasChildren">
        <i :class="['fa fa-fw', {'fa-compress': !collapsed, 'fa-expand': collapsed}]" v-on:click.stop="toggleCollapsed()" title="Collpase or expand child nodes"></i>
      </div>
      <div class="plan-node-body card"
           @mouseenter="eventBus.$emit('mouseovernode', node.nodeId)"
           @mouseleave="eventBus.$emit('mouseoutnode', node.nodeId)"
      >
        <div :class="['card-body header no-focus-outline', taskClassName]"
            v-on:click.stop="showDetails = !showDetails"
        >
          <header class="mb-0">
            <h4 class="text-body">
              <span class="font-weight-normal small">#{{node.nodeId}}</span>
              {{ getNodeName() }}
            </h4>
            <div class="float-right">
              <span v-if="durationClass" :class="'p-0  d-inline-block mb-0 ml-1 text-nowrap alert ' + durationClass" content="Slow" v-tippy><i class="fa fa-fw fa-clock"></i></span>
              <span v-if="costClass" :class="'p-0  d-inline-block mb-0 ml-1 text-nowrap alert ' + costClass" content="Cost is high" v-tippy><i class="fa fa-fw fa-dollar-sign"></i></span>
              <span v-if="estimationClass" :class="'p-0  d-inline-block mb-0 ml-1 text-nowrap alert ' + estimationClass" content="Bad estimation for number of rows" v-tippy><i class="fa fa-fw fa-thumbs-down"></i></span>
              <span v-if="rowsRemovedClass" :class="'p-0  d-inline-block mb-0 ml-1 text-nowrap alert ' + rowsRemovedClass" :content="filterTooltip" v-tippy><i class="fa fa-fw fa-filter"></i></span>
              <span v-if="heapFetchesClass" :class="'p-0  d-inline-block mb-0 ml-1 text-nowrap alert ' + heapFetchesClass" content="Heap Fetches number is high" v-tippy="{arrow: true}"><i class="fa fa-fw fa-exchange-alt"></i></span>
              <span v-if="rowsRemoved && !rowsRemovedClass" class="p-0  d-inline-block mb-0 ml-1 text-nowrap" :content="filterTooltip" v-tippy><i class="fa fa-fw fa-filter text-muted"></i></span>
            </div>
            <span v-if="viewOptions.viewMode === viewModes.FULL">
              <span class="node-duration text-warning" v-if="isNeverExecuted">
                Never executed
              </span>
            </span>
          </header>

          <div v-if="viewOptions.viewMode === viewModes.FULL" class="text-left text-monospace">
            <div v-if="node[nodeProps.RELATION_NAME]" :class="{'line-clamp-2': !showDetails}">
              <span class="text-muted">on&nbsp;</span><span v-if="node[nodeProps.SCHEMA]">{{node[nodeProps.SCHEMA]}}.</span>{{node[nodeProps.RELATION_NAME]}}
              <span v-if="node[nodeProps.ALIAS]">
                <span class="text-muted">as</span>
                {{node[nodeProps.ALIAS]}}
              </span>
            </div>
            <div v-if="node[nodeProps.GROUP_KEY]" :class="{'line-clamp-2': !showDetails}">
              <span class="text-muted">by</span>&nbsp;<span v-html="$options.filters.keysToString(node[nodeProps.GROUP_KEY])"></span></div>
            <div v-if="node[nodeProps.SORT_KEY]" :class="{'line-clamp-2': !showDetails}">
              <span class="text-muted">by</span> <span v-html="$options.filters.sortKeys(node[nodeProps.SORT_KEY], node[nodeProps.PRESORTED_KEY])"></span>
            </div>
            <div v-if="node[nodeProps.JOIN_TYPE]">{{node[nodeProps.JOIN_TYPE] }}
              <span class="text-muted">join</span></div>
            <div v-if="node[nodeProps.INDEX_NAME]" :class="{'line-clamp-2': !showDetails}">
              <span class="text-muted">using</span>&nbsp;<span v-html="$options.filters.keysToString(node[nodeProps.INDEX_NAME])"></span>
            </div>
            <div v-if="node[nodeProps.HASH_CONDITION]" :class="{'line-clamp-2': !showDetails}">
              <span class="text-muted">on</span>&nbsp;<span v-html="$options.filters.keysToString(node[nodeProps.HASH_CONDITION])"></span></div>
            <div v-if="node[nodeProps.CTE_NAME]">
              <a class="text-reset" href v-on:click.stop.prevent="eventBus.$emit('clickcte', 'CTE ' + node[nodeProps.CTE_NAME])">
                <i class="fa fa-search text-muted"></i>&nbsp;<span class="text-muted">CTE</span> {{node[nodeProps.CTE_NAME]}}
              </a>
            </div>
          </div>

          <div v-if="!allWorkersLaunched && viewOptions.viewMode === viewModes.FULL" class="text-c-3 cursor-help" :content="getHelpMessage('workers planned not launched')" v-tippy>
            <i class="fa fa-exclamation-triangle"></i>&nbsp;
            <span>Not all workers launched</span>
          </div>
          <div class="clearfix"></div>

          <div v-if="viewOptions.highlightType !== highlightTypes.NONE && highlightValue !== null">
            <div class="progress node-bar-container" style="height: 5px;">
              <div class="progress-bar" role="progressbar" v-bind:style="{ width: barWidth + '%', 'background-color': getBarColor(barWidth)}" aria-valuenow="0" aria-valuemin="0" aria-valuemax="100"></div>
            </div>
            <span class="node-bar-label" v-if="shouldShowNodeBarLabel()">
              <span class="text-muted">{{viewOptions.highlightType}}:</span>&nbsp;
              <span v-html="highlightValue"></span>
            </span>
          </div>

        </div>

        <div v-if="showDetails" class="card-header border-top">
          <div v-if="getNodeTypeDescription()" class="node-description">
            <span class="node-type">{{node[nodeProps.NODE_TYPE]}} Node</span>&nbsp;<span v-html="getNodeTypeDescription()"></span>
          </div>
          <ul class="nav nav-tabs card-header-tabs">
            <li class="nav-item">
              <a class="nav-link" :class="{'active' : activeTab === 'general' }" @click.prevent="setActiveTab('general')" href>General</a>
            </li>
            <li class="nav-item">
              <a class="nav-link" :class="{'active' : activeTab === 'execution-info' }" @click.prevent="setActiveTab('execution-info')" href>Execution Info</a>
            </li>
            <li class="nav-item">
              <a class="nav-link" :class="{'active' : activeTab === 'misc' }" @click.prevent="setActiveTab('misc')" href>Misc</a>
            </li>
            <li class="nav-item" v-if="costClass === 'c-4'">
              <a class="nav-link" :class="{'active' : activeTab === 'advisor' }" @click.prevent="setActiveTab('advisor')" href>Advisor</a>
            </li>
          </ul>
        </div>
        <div class="card-body tab-content" v-if="showDetails">
          <div class="tab-pane" :class="{'show active': activeTab === 'general' }">
            <!-- general -->
            <div v-if="plan.isAnalyze">
              <i class="fa fa-fw fa-clock text-muted"></i>
              <b>Timing:</b>&nbsp;
              <span :class="'p-0 px-1 rounded alert ' + durationClass" v-html="formattedProp('EXCLUSIVE_DURATION')"></span>
              <template v-if="executionTimePercent !== Infinity">
                |
                <strong>{{executionTimePercent}}</strong><span class="text-muted">%</span>
              </template>
            </div>
            <div>
              <i class="fa fa-fw fa-align-justify text-muted"></i>
              <b>Rows:</b> <span class="px-1">{{ tilde + formattedProp('ACTUAL_ROWS_REVISED') }}</span> <span class="text-muted" v-if="node[nodeProps.PLAN_ROWS]">(Planned: {{ tilde + formattedProp('PLAN_ROWS_REVISED') }})</span>
              <span v-if="plannerRowEstimateDirection !== estimateDirections.none && shouldShowPlannerEstimate">
                |
                <span v-if="plannerRowEstimateDirection === estimateDirections.over"><i class="fa fa-arrow-up"></i> over</span>
                <span v-if="plannerRowEstimateDirection === estimateDirections.under"><i class="fa fa-arrow-down"></i> under</span>
                estimated
                <span v-if="plannerRowEstimateValue != Infinity"> by
                  <span :class="'p-0 px-1 alert ' + estimationClass" v-html="formattedProp('PLANNER_ESTIMATE_FACTOR')"></span>
                </span>
              </span>
            </div>
            <div v-if="rowsRemoved">
              <i class="fa fa-fw fa-filter text-muted"></i>
              <b>
                {{ nodeProps[rowsRemovedProp] }}:
              </b>
              <span>
                <span class="px-1">{{ tilde + formattedProp(rowsRemovedProp) }}</span>|
                <span :class="'p-0 px-1 alert ' + rowsRemovedClass">{{ rowsRemovedPercentString }}%</span>
              </span>
            </div>
            <div v-if="node[nodeProps.HEAP_FETCHES]">
              <i class="fa fa-fw fa-exchange-alt text-muted"></i>
              <b>Heap Fetches:</b>&nbsp;
              <span :class="'p-0 px-1 rounded alert ' + heapFetchesClass" v-html="formattedProp('HEAP_FETCHES')"></span>
              &nbsp;
              <i class="fa fa-fw fa-info-circle text-muted" v-if="heapFetchesClass"
                content="Visibility map may be out-of-date. Consider using VACUUM or change autovacuum settings."
                v-tippy="{arrow: true}"
              ></i>
            </div>
            <div v-if="node[nodeProps.EXCLUSIVE_COST]">
              <i class="fa fa-fw fa-dollar-sign text-muted"></i>
              <b>Cost:</b> <span :class="'p-0 px-1 mr-1 alert ' + costClass">{{ formattedProp('EXCLUSIVE_COST') }}</span> <span class="text-muted">(Total: {{ formattedProp('TOTAL_COST') }})</span>
            </div>
            <div v-if="node[nodeProps.ACTUAL_LOOPS] > 1">
              <i class="fa fa-fw fa-undo text-muted"></i>
              <b>Loops:</b> <span class="px-1">{{ formattedProp('ACTUAL_LOOPS') }}
              </span>
            </div>

            <!-- TIDB Custom statistic -->
            <div v-if="node[nodeProps.DISK]" title="Disk Usage">
              <i class="fa-hdd fa fa-fw text-muted"></i>
              <span>
                <b>Disk:</b> <span class="px-1">{{ formattedProp('DISK') }}</span>
              </span>
            </div>

             <div v-if="node[nodeProps.MEMORY]" title="Memory Usage">
              <i class="fa-memory fa fa-fw text-muted"></i>
              <span>
                <b>Memory:</b> <span class="px-1">{{ formattedProp('MEMORY') }}</span>
              </span>
            </div>

             <!-- <div v-if="node[nodeProps.TASK]" title="Task">
              <i class="fa-thumbtack fa fa-fw text-muted"></i>
              <span>
                <b>Task:</b> <span class="px-1">{{ formattedProp('TASK') }}</span>
              </span>
            </div> -->

            <!-- general tab -->
          </div>
          <div class="tab-pane overflow-auto text-monospace" :class="{'show active': activeTab === 'output' }" v-html="formattedProp('OUTPUT')" style="max-height: 200px">
            <!-- output tab -->
          </div>
          <div class="tab-pane" :class="{'show active': activeTab === 'workers' }" v-if="node[nodeProps.WORKERS_PLANNED] || node[nodeProps.WORKERS_PLANNED_BY_GATHER]">
            <!-- workers tab -->
            <div v-if="(node[nodeProps.WORKERS_PLANNED] || node[nodeProps.WORKERS_PLANNED_BY_GATHER]) && viewOptions.viewMode === viewModes.FULL">
              <b>Workers planned: </b> <span class="px-1">{{ node[nodeProps.WORKERS_PLANNED] || node[nodeProps.WORKERS_PLANNED_BY_GATHER] }}</span>
              <em v-if="!node[nodeProps.WORKERS_PLANNED] && !node[nodeProps.WORKERS] && (!plan.isVerbose || !plan.isAnalyze)" class="text-warning">
                <i class="fa fa-exclamation-triangle cursor-help" :content="getHelpMessage('fuzzy needs verbose')" v-tippy></i>
              </em>
            </div>
            <div v-if="node[nodeProps.WORKERS_LAUNCHED] && viewOptions.viewMode === viewModes.FULL">
              <b>Workers launched: </b> <span class="px-1">{{ node[nodeProps.WORKERS_LAUNCHED] }}</span>
            </div>
            <div v-if="!workersLaunchedCount && node[nodeProps.WORKERS_PLANNED_BY_GATHER]" class="text-muted">
              <em>
                Detailed information is not available.
                  <i class="fa fa-info-circle cursor-help" :content="getHelpMessage('workers detailed info missing')" v-tippy></i>
              </em>
            </div>

            <div class="accordion" v-if="lodash.isArray(node[nodeProps.WORKERS])">
              <template v-for="(worker, index) in node[nodeProps.WORKERS]">
                <div class="card">
                  <div class="card-header p-0">
                    <button class="btn btn-link btn-sm text-secondary" type="button" data-toggle="collapse" :data-target="'#collapse-' + _uid + '-' + index" style="font-size: inherit;">
                      <i class="fa fa-chevron-right fa-fw"></i>
                      <i class="fa fa-chevron-down fa-fw"></i>
                      Worker {{ worker[workerProps.WORKER_NUMBER] }}
                    </button>
                  </div>

                  <div :id="'collapse-' + _uid + '-' + index" class="collapse">
                    <div class="card-body p-0">
                      <table class="table table-sm prop-list mb-0">
                        <tr v-for="(value, key) in worker" v-if="shouldShowProp(key, value)">
                          <td width="40%">{{key}}</td>
                          <td v-html="$options.filters.formatNodeProp(key, value, true)"></td>
                        </tr>
                      </table>
                    </div>
                  </div>
                </div>
              </template>
            </div>
            <!-- workers tab -->
          </div>
          <div class="tab-pane" :class="{'show active': activeTab === 'execution-info'}">
              <!-- Execution info tab -->
              <div>
                {{node[nodeProps.EXECUTION_INFO]}}
              </div>
          </div>
          <div class="tab-pane" :class="{'show active': activeTab === 'misc'}">
            <!-- misc tab -->
            <table class="table table-sm prop-list">
              <tr v-for="prop in props" v-if="shouldShowProp(prop.key, prop.value)">
                <td width="40%">{{prop.key}}</td>
                <td v-html="$options.filters.formatNodeProp(prop.key, prop.value, true)"></td>
              </tr>
            </table>

            <div class="text-muted text-right"><em>* Calculated value</em></div>
            <!-- misc tab -->
          </div>
          <div class="tab-pane" :class="{'show active': activeTab === 'advisor'}">
              <!-- Advisor tab -->
              <div>
                <table class="table table-sm prop-list">
                  <tr>
                    <td width="40%">Hint: </td>
                    <td>USE_INDEX</td>
                  </tr>
                  <tr>
                    <td width="40%">Original Cost:</td>
                    <td>{{ formattedProp('EXCLUSIVE_COST') }}</td>
                  </tr>
                  <tr>
                    <td>Tunning Cost:</td>
                    <td>{{'To be added'}}</td>
                  </tr>
                </table>
                <button class="btn btn-primary btn-small float-right">Apply</button>
              </div>
          </div>
        </div>

      </div>
    </div>
    <ul v-if="plans" :class="['node-children', {'collapsed': collapsed}]">
      <li v-for="subnode in plans">
        <plan-node :node="subnode" :plan="plan" :viewOptions="viewOptions" :eventBus="eventBus">
          <template v-slot:nodeindex="{ node }">
            <slot name="nodeindex" v-bind:node="node"></slot>
          </template>
        </plan-node>
      </li>
    </ul>
  </div>
</template>

<script lang="ts">
import { Component, Prop, Vue, Watch } from 'vue-property-decorator';
import { HelpService } from '@/services/help-service';
import { ColorService } from '@/services/color-service';
import { cost, duration, factor, formatNodeProp, keysToString, sortKeys, truncate, rows } from '@/filters';
import { EstimateDirection, HighlightType, NodeProp, nodePropTypes, Orientation,
         PropType, ViewMode, WorkerProp } from '@/enums';
import * as _ from 'lodash';
import { includes } from 'lodash';

@Component({
  name: 'plan-node',
  filters: {
    cost,
    duration,
    factor,
    formatNodeProp,
    keysToString,
    sortKeys,
    truncate,
    rows,
  },
})
export default class PlanNode extends Vue {
  @Prop(Object) public node!: any;
  public executionTimePercent: number = NaN;
  public selected: boolean = false;
  @Prop(Object) private plan!: any;
  @Prop(Object) private viewOptions!: any;
  @Prop() private eventBus!: InstanceType<typeof Vue>;
  
  // UI flags
  private showDetails: boolean = false;
  private collapsed: boolean = false;
  private activeTab: string = 'general';

  // calculated properties
  private costPercent: number = NaN;
  private barWidth: number = 0;
  private highlightValue?: string | null;
  private props: any[] = [];
  private plans: any[] = [];
  private plannerRowEstimateValue?: number;
  private plannerRowEstimateDirection?: EstimateDirection;
  private rowsRemoved: number = NaN;
  private rowsRemovedPercent: number = NaN;
  private rowsRemovedPercentString?: string | null;

  // required for custom change detection
  private currentCompactView?: boolean;

  // expose enum to view
  private estimateDirections = EstimateDirection;
  private highlightTypes = HighlightType;
  private viewModes = ViewMode;
  private orientations = Orientation;

  private nodeProps = NodeProp;
  private workerProps = WorkerProp;
  private helpService = new HelpService();
  private colorService = new ColorService();
  private lodash = _;

  // Returns the list of properties that have already been displayed either in
  // the main panel or in other detailed tabs.
  private notMiscProperties: string[] = [
      NodeProp.NODE_TYPE,
      NodeProp.CTE_NAME,
      NodeProp.EXCLUSIVE_DURATION,
      NodeProp.EXECUTION_INFO,
      NodeProp.ACTUAL_TOTAL_TIME,
      NodeProp.MEMORY,
      NodeProp.DISK,
      NodeProp.TOTAL_TIME,
      NodeProp.EXCLUSIVE_COST,
      NodeProp.TOTAL_COST,
      NodeProp.PLAN_ROWS,
      NodeProp.ACTUAL_ROWS,
      NodeProp.ACTUAL_LOOPS,
      NodeProp.OUTPUT,
      NodeProp.WORKERS,
      NodeProp.WORKERS_PLANNED,
      NodeProp.WORKERS_LAUNCHED,
      NodeProp.EXCLUSIVE_SHARED_HIT_BLOCKS,
      NodeProp.EXCLUSIVE_SHARED_READ_BLOCKS,
      NodeProp.EXCLUSIVE_SHARED_DIRTIED_BLOCKS,
      NodeProp.EXCLUSIVE_SHARED_WRITTEN_BLOCKS,
      NodeProp.EXCLUSIVE_TEMP_READ_BLOCKS,
      NodeProp.EXCLUSIVE_TEMP_WRITTEN_BLOCKS,
      NodeProp.EXCLUSIVE_LOCAL_HIT_BLOCKS,
      NodeProp.EXCLUSIVE_LOCAL_READ_BLOCKS,
      NodeProp.EXCLUSIVE_LOCAL_DIRTIED_BLOCKS,
      NodeProp.EXCLUSIVE_LOCAL_WRITTEN_BLOCKS,
      NodeProp.SHARED_HIT_BLOCKS,
      NodeProp.SHARED_READ_BLOCKS,
      NodeProp.SHARED_DIRTIED_BLOCKS,
      NodeProp.SHARED_WRITTEN_BLOCKS,
      NodeProp.TEMP_READ_BLOCKS,
      NodeProp.TEMP_WRITTEN_BLOCKS,
      NodeProp.LOCAL_HIT_BLOCKS,
      NodeProp.LOCAL_READ_BLOCKS,
      NodeProp.LOCAL_DIRTIED_BLOCKS,
      NodeProp.LOCAL_WRITTEN_BLOCKS,
      NodeProp.PLANNER_ESTIMATE_FACTOR,
      NodeProp.PLANNER_ESTIMATE_DIRECTION,
      NodeProp.SUBPLAN_NAME,
      NodeProp.GROUP_KEY,
      NodeProp.HASH_CONDITION,
      NodeProp.JOIN_TYPE,
      NodeProp.INDEX_NAME,
      NodeProp.HASH_CONDITION,
      NodeProp.EXCLUSIVE_IO_READ_TIME,
      NodeProp.EXCLUSIVE_IO_WRITE_TIME,
      NodeProp.IO_READ_TIME, // Exclusive value already shown in IO tab
      NodeProp.IO_WRITE_TIME, // Exclusive value already shown in IO tab
      NodeProp.HEAP_FETCHES,
      NodeProp.WAL_RECORDS,
      NodeProp.WAL_BYTES,
      NodeProp.WAL_FPI,
      NodeProp.NODE_ID,
      NodeProp.ROWS_REMOVED_BY_FILTER,
      NodeProp.ROWS_REMOVED_BY_JOIN_FILTER,
      NodeProp.ACTUAL_ROWS_REVISED,
      NodeProp.PLAN_ROWS_REVISED,
      NodeProp.ROWS_REMOVED_BY_FILTER_REVISED,
      NodeProp.ROWS_REMOVED_BY_JOIN_FILTER_REVISED,
  ];

  public setShowDetails(showDetails: boolean): void {
    this.showDetails = showDetails;
  }

  created() {
    this.handleupdate();
  }

  @Watch('plan')
  private handleupdate(): void {
    this.calculateProps();
    this.calculateBar();
    this.calculateDuration();
    this.calculateCost();
    this.calculateRowsRemoved();

    this.plans = this.node[NodeProp.PLANS];
    this.showDetails = false;

    this.plannerRowEstimateDirection = this.node[NodeProp.PLANNER_ESTIMATE_DIRECTION];
    this.plannerRowEstimateValue = this.node[NodeProp.PLANNER_ESTIMATE_FACTOR];
  }

  private calculateDuration() {
    // use the first node total time if plan execution time is not available
    const executionTime = this.plan.planStats.executionTime || this.plan.content.Plan[NodeProp.ACTUAL_TOTAL_TIME];
    this.executionTimePercent = _.round(
      (this.node[NodeProp.EXCLUSIVE_DURATION] / executionTime) * 100);
  }

  private calculateCost() {
    const maxTotalCost = this.plan.content.maxTotalCost;
    this.costPercent = _.round((this.node[NodeProp.EXCLUSIVE_COST] / maxTotalCost) * 100);
  }

  private get highestCost() {
    const { maxRows } = this.plan.planStats;
    return maxRows;
  }

  private get rowsRemovedProp() {
    const nodeKey = Object.keys(this.node).find(
      (key) => key === NodeProp.ROWS_REMOVED_BY_FILTER_REVISED || key === NodeProp.ROWS_REMOVED_BY_JOIN_FILTER_REVISED,
    );
    type NodePropStrings = keyof typeof NodeProp;
    return Object.keys(NodeProp).find((prop) => NodeProp[prop as NodePropStrings] === nodeKey);
  }

  private calculateRowsRemoved() {
    const rowsRemovedProp = this.rowsRemovedProp;

    if (rowsRemovedProp) {
      type NodePropStrings = keyof typeof NodeProp;
      const removed = this.node[NodeProp[rowsRemovedProp as NodePropStrings]];
      this.rowsRemoved = removed;
      const actual = this.node[NodeProp.ACTUAL_ROWS];
      this.rowsRemovedPercent = _.floor(removed / (removed + actual) * 100);
      if (this.rowsRemovedPercent === 100) {
        this.rowsRemovedPercentString = '>99';
      } else if (this.rowsRemovedPercent === 0) {
        this.rowsRemovedPercentString = '<1';
      } else {
        this.rowsRemovedPercentString = this.rowsRemovedPercent.toString();
      }
    }
  }

  // create an array of node propeties so that they can be displayed in the view
  private calculateProps() {
    this.props = _.chain(this.node)
      .omit(NodeProp.PLANS)
      .omit(NodeProp.WORKERS)
      .map((value, key) => {
        return { key, value };
      })
      .value();
  }

  private getNodeTypeDescription() {
    return this.helpService.getNodeTypeDescription(this.node[NodeProp.NODE_TYPE]);
  }

  private getNodeName(): string {
    let nodeName = this.isParallelAware ? 'Parallel ' : '';
    nodeName += this.node[NodeProp.NODE_TYPE];
    nodeName = nodeName.replace(/_\d*/g, ' ');
    if (this.viewOptions.viewMode === ViewMode.DOT && !this.showDetails) {
      return nodeName.replace(/[^A-Z]/g, '');
    }
    return nodeName;
  }

  private get shouldShowPlannerEstimate() {
    if ((this.collapsed && !this.showDetails) || this.viewOptions.viewMode === ViewMode.DOT) {
      return false;
    }
    return (this.estimationClass || this.showDetails) &&
      this.plannerRowEstimateDirection !== EstimateDirection.none &&
      this.plannerRowEstimateValue;
  }

  private get shouldShowCost() {
    if ((this.collapsed && !this.showDetails) || this.viewOptions.viewMode === ViewMode.DOT) {
      return false;
    }

    return this.node[NodeProp.EXCLUSIVE_COST] && (this.costClass || this.showDetails);
  }

  private shouldShowNodeBarLabel(): boolean {
    if (this.showDetails) {
      return true;
    }

    if (this.collapsed || this.viewOptions.viewMode === ViewMode.DOT) {
      return false;
    }

    return true;
  }

  private get shouldShowRowsRemoved(): boolean {
    return !!this.rowsRemovedClass && this.viewOptions.viewMode === ViewMode.FULL && !this.collapsed;
  }

  @Watch('viewOptions.highlightType')
  private calculateBar(): void {
    let value: number;
    if (!this.$options || !this.$options.filters) {
      return;
    }
    switch (this.viewOptions.highlightType) {
      case HighlightType.DURATION:
        value = (this.node[NodeProp.EXCLUSIVE_DURATION]);
        if (value === undefined) {
          this.highlightValue = null;
          break;
        }
        this.barWidth = Math.round(value / this.plan.planStats.maxDuration * 100);
        this.highlightValue = this.$options.filters.duration(value);
        break;
      case HighlightType.ROWS:
        value = (this.node[NodeProp.ACTUAL_ROWS]);
        if (value === undefined) {
          this.highlightValue = null;
          break;
        }
        this.barWidth = Math.round(value / this.plan.planStats.maxRows * 100) || 0;
        this.highlightValue = this.$options.filters.rows(value);
        break;
      case HighlightType.COST:
        value = (this.node[NodeProp.EXCLUSIVE_COST]);
        if (value === undefined) {
          this.highlightValue = null;
          break;
        }
        this.barWidth = Math.round(value / this.plan.planStats.maxCost * 100);
        this.highlightValue = this.$options.filters.cost(value);
        break;
    }
  }

  private getBarColor(percent: number) {
    return this.colorService.numberToColorHsl(percent);
  }

  private get durationClass() {
    let c;
    const i = this.executionTimePercent;
    
    if (i > 90) {
      return false;
    } else if (i > 40) {
      c = 3;
    } else if (i > 10) {
      c = 2;
    }
    if (c) {
      return 'c-' + c;
    }
    return false;
  }

  private get estimationClass() {
    let c;
    const i = this.node[NodeProp.PLANNER_ESTIMATE_FACTOR];
    if (i > 1000) {
      c = 4;
    } else if (i > 100) {
      c = 3;
    } else if (i > 10) {
      c = 2;
    }
    if (c) {
      return 'c-' + c;
    }
    return false;
  }

  private get costClass() {
    let c;
    const i = this.costPercent;
    if (i > 90) {
      c = 4;
    } else if (i > 40) {
      c = 3;
    } else if (i > 10) {
      c = 2;
    }
    if (c) {
      return 'c-' + c;
    }
    return false;
  }

  private get rowsRemovedClass() {
    let c;
    // high percent of rows removed is relevant only when duration is high
    // as well
    const i = this.rowsRemovedPercent * this.executionTimePercent;
    if (i > 2000) {
      c = 4;
    } else if (i > 500) {
      c = 3;
    }
    if (c) {
      return 'c-' + c;
    }
    return false;
  }

  private get heapFetchesClass() {
    let c;
    const i = this.node[NodeProp.HEAP_FETCHES] /
      (this.node[NodeProp.ACTUAL_ROWS] +
       (this.node[NodeProp.ROWS_REMOVED_BY_FILTER] || 0) +
       (this.node[NodeProp.ROWS_REMOVED_BY_JOIN_FILTER] || 0)
      ) * 100;
    if (i > 90) {
      c = 4;
    } else if (i > 40) {
      c = 3;
    } else if (i > 10) {
      c = 2;
    }
    if (c) {
      return 'c-' + c;
    }
    return false;
  }

  private toggleCollapsed() {
    this.collapsed = !this.collapsed;
  }

  private get hasChildren(): boolean {
    return !!this.plans;
  }

  private get filterTooltip(): string {
    return this.rowsRemovedPercentString + '% of rows removed by filter';
  }

  private get workersLaunchedCount(): number {
    if (_.isArray(this.node[NodeProp.WORKERS])) {
      return this.node[NodeProp.WORKERS].length;
    }
    return parseInt(this.node[NodeProp.WORKERS], 0);
  }

  private get workersPlannedCount(): number {
    return this.node[NodeProp.WORKERS_PLANNED_BY_GATHER];
  }

  private get workersPlannedCountReversed(): number[] {
    const workersPlanned = this.node[NodeProp.WORKERS_PLANNED_BY_GATHER];
    return [...Array(workersPlanned).keys()].slice().reverse();
  }

  private get isParallelAware(): boolean {
    return this.node[NodeProp.PARALLEL_AWARE];
  }

  private get allWorkersLaunched(): boolean {
    return !this.node[NodeProp.WORKERS_LAUNCHED] ||
      this.node[NodeProp.WORKERS_PLANNED] === this.node[NodeProp.WORKERS_LAUNCHED];
  }

  private getHelpMessage(message: string) {
    return this.helpService.getHelpMessage(message);
  }

  private shouldShowProp(key: string, value: any): boolean {

    return (value ||
            nodePropTypes[key] === PropType.increment ||
            key === NodeProp.ACTUAL_ROWS) &&
           this.notMiscProperties.indexOf(key) === -1;
  }

  private get shouldShowIoBuffers(): boolean {
    const properties: Array<keyof typeof NodeProp> = [
      'EXCLUSIVE_SHARED_HIT_BLOCKS',
      'EXCLUSIVE_SHARED_READ_BLOCKS',
      'EXCLUSIVE_SHARED_DIRTIED_BLOCKS',
      'EXCLUSIVE_SHARED_WRITTEN_BLOCKS',
      'EXCLUSIVE_TEMP_READ_BLOCKS',
      'EXCLUSIVE_TEMP_WRITTEN_BLOCKS',
      'EXCLUSIVE_LOCAL_HIT_BLOCKS',
      'EXCLUSIVE_LOCAL_READ_BLOCKS',
      'EXCLUSIVE_LOCAL_DIRTIED_BLOCKS',
      'EXCLUSIVE_LOCAL_WRITTEN_BLOCKS',
      'EXCLUSIVE_IO_READ_TIME',
      'EXCLUSIVE_IO_WRITE_TIME',
    ];
    const values = _.map(properties, (property) => {
      const value = this.node[NodeProp[property]];
      return _.isNaN(value) ? 0 : value;
    });
    const sum = _.sum(values);
    return sum > 0;
  }

  private get isNeverExecuted(): boolean {
    return this.plan.planStats.executionTime && !this.node[NodeProp.ACTUAL_LOOPS];
  }

  private get hasSeveralLoops(): boolean {
    return this.node[NodeProp.ACTUAL_LOOPS] > 1;
  }

  private get tilde(): string {
    return this.hasSeveralLoops ? '~' : '';
  }

  private setActiveTab(tab: string) {
    this.activeTab = tab;
  }

  // returns the formatted prop
  private formattedProp(propName: keyof typeof NodeProp) {
    const property = NodeProp[propName];
    const value = this.node[property];
    return this.$options!.filters!.formatNodeProp(property, value);
  }

  private get taskClassName() {
    const task = this.node[NodeProp.TASK];
    if (!task) {
      return '';
    }
    if (task === 'root') {
      return 'task-tidb';
    }
    if (task.includes('tikv')) {
      return 'task-tikv';
    }
    if (task.includes('tiflash')) {
      return 'task-tiflash';
    }
  }
}
</script>
