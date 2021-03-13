<template lang="pug">
div
  div#graph.visualization
  div#timeline.visualization
</template>

<style lang="scss">
div.visualization {
  margin-top: 0.5em;
  margin-bottom: 0.5em;

  .timeline-timeline {
    font-family: sans-serif !important;

    .timeline-panel {
      box-sizing: border-box;
    }

    .timeline-item {
      border-radius: 2px;
    }
  }
  .vis-label {
    width: 200px;
  }
  .vis-panel.vis-left {
    width: 200px;
  }
  /* https://github.com/visjs/vis-timeline/issues/218 */
  .vis-panel.vis-background.vis-horizontal .vis-grid.vis-minor {
    margin-left: 124px;
  }
  .vis-panel.vis-background.vis-horizontal .vis-grid.vis-major {
    margin-left: 126px;
  }
}
</style>

<script lang="ts">
import _ from 'lodash';
import moment from 'moment';
import Color from 'color';
import { buildTooltip } from '../util/tooltip.js';
import { getColorFromString, getTitleAttr } from '../util/color';

import { Timeline, Graph2d } from 'vis-timeline/esnext';
import 'vis-timeline/styles/vis-timeline-graph2d.css';

export default {
  props: {
    buckets: { type: Array },
    showRowLabels: { type: Boolean },
    queriedInterval: { type: Array },
    showQueriedInterval: { type: Boolean },
  },
  data() {
    return {
      timeline: null,
      filterShortEvents: true,
      options: {
        zoomMin: 1000 * 60, // 10min in milliseconds
        zoomMax: 1000 * 60 * 60 * 24 * 31 * 3, // about three months in milliseconds
        stack: false,
        tooltip: {
          followMouse: true,
          overflowMethod: 'cap',
        },
      },
    };
  },
  computed: {
    chartData() {
      const data = [];
      _.each(this.buckets, (bucket, bidx) => {
        if (bucket.events === undefined) {
          return;
        }
        let events = bucket.events;
        // Filter out events shorter than 1 second (notably including 0-duration events)
        // TODO: Use flooding instead, preferably with some additional method of removing/simplifying short events for even greater performance
        if (this.filterShortEvents) {
          events = _.filter(events, e => e.duration > 1);
        }
        events = _.sortBy(events, e => e.timestamp);
        _.each(events, e => {
          data.push([
            bidx,
            getTitleAttr(bucket, e),
            buildTooltip(bucket, e),
            new Date(e.timestamp),
            new Date(moment(e.timestamp).add(e.duration, 'seconds').valueOf()),
            getColorFromString(getTitleAttr(bucket, e)),
          ]);
        });
      });
      return data;
    },
  },
  watch: {
    buckets() {
      // For some reason, an object is passed here, after which the correct array arrives
      if (this.buckets.length === undefined) {
        //console.log("I told you so!")
        return;
      }
      console.log("buckets");
      console.log(this.buckets);

      // Build groups
      const groups = _.map(this.buckets, (bucket, bidx) => {
        return { id: bidx, content: this.showRowLabels ? bucket.id : '' };
      });

      // Build items
      const items = _.map(this.chartData, (row, i) => {
        const bgColor = row[5];
        const borderColor = Color(bgColor).darken(0.3);
        return {
          id: String(i),
          group: row[0],
          content: row[1],
          title: row[2],
          start: moment(row[3]),
          end: moment(row[4]),
          style: `background-color: ${bgColor}; border-color: ${borderColor}`,
        };
      });

      if (groups.length > 0 && items.length > 0) {
        if (this.queriedInterval && this.showQueriedInterval) {
          const duration = this.queriedInterval[1].diff(this.queriedInterval[0], 'seconds');
          groups.push({ id: String(groups.length), content: 'queried interval' });
          items.push({
            id: String(items.length + 1),
            group: groups.length - 1,
            title: buildTooltip(
              { type: 'test' },
              {
                timestamp: this.queriedInterval[0],
                duration: duration,
                data: { title: 'test' },
              }
            ),
            content: 'query',
            start: this.queriedInterval[0],
            end: this.queriedInterval[1],
            style: 'background-color: #aaa; height: 10px',
          });
        }

        const start =
          (this.queriedInterval && this.queriedInterval[0]) ||
          _.min(_.map(items, item => item.start));
        const end =
          (this.queriedInterval && this.queriedInterval[1]) ||
          _.max(_.map(items, item => item.end));
        this.options.min = start;
        this.options.max = end;
        this.timeline.setOptions(this.options);
        this.graph.setOptions(this.options);
        this.timeline.setWindow(start, end);
        this.graph.setWindow(start, end);
        this.timeline.setData({ groups: groups, items: items });
        const bin_width = 30; // seconds
        let cur_bin_start = moment(items[0].start);
        const graph_items = [{x: cur_bin_start, y: 0}];
        items.slice(1).forEach(item => {
          const cur_bin_end = cur_bin_start.clone().add(bin_width, 'second');
          if (item.start > cur_bin_end) {
            graph_items.push({
              x: cur_bin_start,
              y: 0,
            });
            console.log(cur_bin_start);
            console.log(cur_bin_end);
            cur_bin_start = cur_bin_end;
          }
          graph_items.slice(-1)[0].y += 1;
        });
        console.log(items);
        console.log(graph_items);
        // https://ww3.arb.ca.gov/ei/tools/lib/vis/docs/graph2d.html#items
        this.graph.setItems(graph_items);
      }
    },
  },
  mounted() {
    this.$nextTick(function () {
      const graph = new Graph2d(this.$el.querySelector('#graph'), [], [], {drawPoints: false});
      const timeline = new Timeline(this.$el.querySelector('#timeline'), [], [], this.options);
      function onChangeGraph(range) {
        if (!range.byUser) {
          return;
        }
        timeline.setOptions({
          start: range.start,
          end: range.end,
        });
      }
      graph.on('rangechange', onChangeGraph);

      function onChangeTimeline(range) {
        if (!range.byUser) {
          return;
        }
        graph.setOptions({
          start: range.start,
          end: range.end,
        });
      }
      timeline.on('rangechange', onChangeTimeline);

      // Need to bind to "this" on separate lines so that we can use the
      // graph/timeline consts for closures in the onChange functions.
      this.graph = graph;
      this.timeline = timeline;

      // // Vis same width label.
      // function visLabelSameWidth() {
      //   const ylabel_width = $('#visualization-bottom-row .vis-labelset .vis-label').width() + 'px';
      //   //$('#visualization-top-row')[0].childNodes[0].childNodes[2].style.left = ylabel_width;

      //   const w1 = $('#visualization-top-row .vis-content .vis-data-axis').width();
      //   const w2 = $('#visualization-bottom-row .vis-labelset .vis-label').width();

      //   // $('#visualization-top-row')[0].childNodes[0].childNodes[2].style.display = 'none';

      //   if (w2 > w1) {
      //     $('#visualization-top-row .vis-content')[1].style.width = ylabel_width;
      //   }
      //   else {
      //     $('#visualization-bottom-row .vis-labelset .vis-label').width(w1 + 'px');
      //   }
      // }
      // this.graph.on('_change', function() {
      //   visLabelSameWidth();
      // });

      // $(window).resize(function(){
      //   visLabelSameWidth();
      // });
    });
  },
};
</script>
