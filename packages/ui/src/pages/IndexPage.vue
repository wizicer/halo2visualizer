<template>
  <q-page class="">
    <div class="q-pa-md">
      <div>
        <q-btn-toggle
          v-model="modelSelection"
          push
          toggle-color="primary"
          :options="dataList.map((_) => ({ label: _.name, value: _ }))"
        />
      </div>
      <q-card v-if="modelSelection" class="">
        <q-card-section>
          <div v-if="modelSelection?.title" class="text-h6">
            {{ modelSelection.title }}
          </div>
        </q-card-section>

        <q-card-section v-if="modelSelection?.description"
          >{{ modelSelection.description }}
        </q-card-section>

        <q-separator v-if="modelSelection?.sourceUrl" />

        <q-card-actions>
          <q-btn
            flat
            v-if="modelSelection?.sourceUrl"
            :href="modelSelection.sourceUrl"
            target="_blank"
            :icon="matOpenInNew"
            >See Source Code</q-btn
          >
        </q-card-actions>
      </q-card>
    </div>
    <div class="q-pa-md">
      <q-checkbox v-model="showTooltip" label="Show Tooltip" />
      <q-checkbox
        v-model="showConstraints"
        label="Show Constraints"
        @click="toggleConstraints()"
      />

      <table>
        <tr v-for="(c, k) in rmapcolor" :key="k">
          <td :style="`border: 1px solid ${c};`">
            {{ k }}
          </td>
        </tr>
      </table>
    </div>
    <div class="q-pa-md row">
      <q-table
        :rows="rows"
        :columns="columns"
        row-key="name"
        flat
        bordered
        dense
        :pagination="pagination"
        :hide-pagination="true"
      >
        <template v-slot:body-cell="props">
          <q-td
            :props="props"
            :class="
              'bg-' +
              getColorByColName(props.col.name) +
              ' ' +
              getBorderOfRegion(props.value, props.col)
            "
            :style="'border-color: ' + rmapcolor[props.value.region] + ';'"
          >
            <q-badge
              :color="getColorByType(props.value.type, props.value.value)"
              :label="props.value.type == 'Selector' ? '' : props.value.value"
              :ref="
                (el) => {
                  const elel = (el as any)?.$el;
                  if (!elel) return;
                  const col = props.col.name;
                  if (!cellBadges[col]) cellBadges[col] = {};
                  cellBadges[col][props.value.index] = elel;
                }
              "
            >
              <q-tooltip :delay="showTooltip ? 0 : 100000">
                {{
                  `${
                    props.value.region ? `Region: ${props.value.region}\n` : ''
                  }Raw: ${props.value.raw}`
                }}
              </q-tooltip>
            </q-badge>
          </q-td>
        </template>
        <template v-slot:header-cell="props">
          <q-th
            :props="props"
            :class="'bg-' + getColorByColName(props.col.name)"
          >
            {{ props.col.label }}
          </q-th>
        </template>
      </q-table>
    </div>
  </q-page>
</template>

<script setup lang="ts">
import { Ref, ref, watch } from 'vue';
import { QTableColumn } from 'quasar';
import { matOpenInNew } from '@quasar/extras/material-icons';
import LeaderLine from 'leader-line-new';
import {
  RowFieldType,
  getColumnDefinition,
  getColumns,
  MockProverData,
  getRowsAndRegions,
  RowFieldWithPosition,
  getPermutationLines,
} from 'src/services/ConstraintSystem';
import { IDataModel, dataList } from 'src/services/DefaultModels';

const modelSelection: Ref<IDataModel | undefined> = ref(undefined);
function getColorByColName(col: string): string {
  col = col.slice(0, col.indexOf('-'));
  return col == 'instance'
    ? 'grey'
    : col == 'advice'
    ? 'deep-orange-3'
    : col == 'fixed'
    ? 'light-blue-2'
    : col == 'selector'
    ? 'indigo-2'
    : col == 'gate'
    ? 'cyan-2'
    : 'negative';
}

function getColorByType(type: RowFieldType, value = ''): string {
  return type == 'Unassigned'
    ? 'grey'
    : type == 'Assigned'
    ? 'teal'
    : type == 'Instance'
    ? 'darkslategray'
    : type == 'Poison'
    ? 'purple'
    : type == 'Selector'
    ? value == 'true'
      ? 'green'
      : 'indigo-3'
    : type == 'Gates'
    ? 'cyan-8'
    : 'negative';
}

const pagination = ref({
  page: 1,
  rowsPerPage: -1,
});
const columns: Ref<QTableColumn[]> = ref([]);

const showTooltip = ref(false);
const showConstraints = ref(false);

function toggleConstraints() {
  for (let i = 0; i < lines.length; i++) {
    const line = lines[i];
    if (showConstraints.value) line.show();
    else line.hide();
  }
}

const rows: Ref<Record<string, RowFieldWithPosition>[]> = ref([]);
const rmap: Ref<Record<string, string>[]> = ref([]);
const rmapcolor: Ref<Record<string, string>> = ref({});

function getBorderOfRegion(
  row: RowFieldWithPosition,
  col: QTableColumn
): string {
  let border = '';
  const y = row.index;
  const x = col.name;
  const name = row.region;
  if (!name) return '';
  if (rmap.value[y - 1] && rmap.value[y - 1][x] == name)
    border += ' no_border_top';
  else border += ' cell_border_top';
  if (rmap.value[y + 1] && rmap.value[y + 1][x] == name)
    border += ' no_border_bottom';
  else border += ' cell_border_bottom';
  const colidx = columns.value.findIndex((_) => _.name == x);
  const left = columns.value[colidx - 1]?.name;
  if (left && rmap.value[y][left] == name) border += ' no_border_left';
  else border += ' cell_border_left';
  const right = columns.value[colidx + 1]?.name;
  if (right && rmap.value[y][right] == name) border += ' no_border_right';
  else border += ' cell_border_right';

  // console.log(border, columns.value, rmap.value, row, col);
  return border;
}

const cellBadges = ref<Record<string, Record<string, Element>>>({});
const lines: LeaderLine[] = [];

function drawLines(data: MockProverData) {
  const plines = getPermutationLines(
    data,
    cellBadges.value,
    columns.value,
    rows.value
  );

  for (let i = 0; i < plines.length; i++) {
    const { fromValue, toValue, from, to } = plines[i];

    const color = fromValue == toValue ? 'wheat' : 'crimson';
    const outlineColor = fromValue == toValue ? 'tan' : 'coral';

    const line = new LeaderLine(
      LeaderLine.mouseHoverAnchor(from as HTMLElement, 'fade', {
        style: {
          backgroundImage: null,
          backgroundColor: null,
          paddingRight: null,
        },
        hoverStyle: {
          backgroundColor: null,
        },
      }),
      to,
      {
        color,
        path: 'straight',
        size: 4,
        outline: true,
        endPlug: 'behind',
        outlineColor,
        dash: { animation: true, gap: 4 },
      }
    );
    lines.push(line);
  }
}

function loadData(data: MockProverData) {
  // console.log(data);
  rows.value = [];
  columns.value = [];

  setTimeout(() => {
    const cols = getColumnDefinition(data);
    const colsdata = getColumns(cols);
    columns.value = colsdata;
    const colorList = ['red', 'blue', 'wheat', 'green'];
    const rr = getRowsAndRegions(data, cols, colorList);
    rows.value = rr.rows;
    rmap.value = rr.rmap;
    rmapcolor.value = rr.rmapcolor;
    for (let i = 0; i < lines.length; i++) {
      const line = lines[i];
      line.remove();
    }
    lines.length = 0;
  }, 100);
  setTimeout(() => drawLines(data), 300);
}

watch(modelSelection, (newValue, oldValue) => {
  if (newValue == oldValue) return;
  if (!newValue) {
    rows.value = [];
    columns.value = [];
    return;
  }
  loadData(newValue.data);
});
modelSelection.value = dataList[0];
</script>

<style scoped lang="scss">
$pos: left, right, top, bottom;

@each $p in $pos {
  .no_border_#{$p} {
    border-#{$p}-width: 0 !important;
  }

  .cell_border_#{$p} {
    border-#{$p}-width: 1px;
  }
}
</style>
