<!-- 
  Copyright Amazon.com, Inc. or its affiliates. All Rights Reserved.
  SPDX-License-Identifier: Apache-2.0
-->

<!--suppress ALL -->
<template>
  <b-container fluid>
    <b-col>
      <b-row
        align-h="center"
        class="my-1"
      >
        <div class="wrapper">
          Confidence Threshold<br>
          <input
            type="range"
            value="90"
            min="55"
            max="99"
            step="1"
            @click="updateConfidence"
          >
          {{ Confidence }}%<br>
        </div>
      </b-row>
      <div v-if="lowerConfidence === true">
        {{ lowerConfidenceMessage }}
      </div>
      <div
        v-if="isBusy"
        class="wrapper"
      >
        <Loading />
      </div>
      <b-row
        align-h="center"
        class="my-1"
      >
        <div class="wrapper">
          <br>
          <template v-for="label in sorted_unique_labels">
            <template v-if="boxes_available.includes(label[0])">
              <!-- Show darker button outline if boxes are available for the label -->
              <b-button
                v-b-tooltip.hover
                variant="outline-dark"
                :title="label[1]"
                size="sm"
                pill
                @click="updateMarkers(label[0])"
              >
                {{ label[0]+"*" }}
              </b-button> &nbsp;
            </template>
            <template v-else>
              <b-button
                v-b-tooltip.hover
                variant="outline-secondary"
                :title="label[1]"
                size="sm"
                pill
                @click="updateMarkers(label[0])"
              >
                {{ label[0] }}
              </b-button> &nbsp;
            </template>
          </template>
        </div>
      </b-row>

      <b-row
        align-h="center"
        class="my-1"
      >
        <div
          v-if="isBusy === false"
          class="wrapper"
        >
          <br><p class="text-muted">
            ({{ count_labels }} identified objects, {{ count_distinct_labels }} unique)
          </p>
          <hr>
          <p class="text-muted">
            * Indicates bounding boxes are available.
          </p>
        </div>
      </b-row>
    </b-col>
    <b-button
      type="button"
      @click="saveFile()"
    >
      Download Data
    </b-button>
    <br>
    <b-button
      :pressed="false"
      size="sm"
      variant="link"
      class="text-decoration-none"
      @click="showElasticsearchApiRequest = true"
    >
      Show API request to get these results
    </b-button>
    <b-modal
      v-model="showElasticsearchApiRequest"
      scrollable
      title="SEARCH API"
      ok-only
    >
      <label>Request URL:</label>
      <pre v-highlightjs><code class="bash">GET {{ SEARCH_ENDPOINT }}workflow/execution</code></pre>
      <label>Search query:</label>
      <pre v-highlightjs="JSON.stringify(searchQuery)"><code class="json"></code></pre>
      <label>Sample command:</label>
      <pre v-highlightjs="curlCommand"><code class="bash"></code></pre>
    </b-modal>
  </b-container>
</template>

<script>
  import { mapState } from 'vuex'
  import Loading from '@/components/Loading.vue'

  export default {
    name: "Celebrities",
    components: {
      Loading
    },
    props: {
      mediaType: {
        type: String,
        default: ""
      },
    },
    data() {
      return {
        curlCommand: '',
        searchQuery: '',
        showElasticsearchApiRequest: false,
        Confidence: 90,
        high_confidence_data: [],
        elasticsearch_data: [],
        boxes_available: [],
        count_distinct_labels: 0,
        count_labels: 0,
        isBusy: false,
        operator: 'celebrity_detection',
        canvasRefreshInterval: undefined,
        timeseries: new Map(),
        selectedLabel: '',
        lowerConfidence: false,
        lowerConfidenceMessage: 'Try lowering confidence threshold'
      }
    },
    computed: {
      ...mapState(['player']),
      sorted_unique_labels() {
        // This function sorts and counts unique labels for mouse over events on label buttons
        const es_data = this.elasticsearch_data;
        const unique_labels = new Map();
        // sort and count unique labels for label mouse over events
        es_data.forEach(function (record) {
          unique_labels.set(record.Name, unique_labels.get(record.Name) ? unique_labels.get(record.Name) + 1 : 1);
          if (record.BoundingBox) {
            // Save this label name to a list of labels that have bounding boxes
            this.saveBoxedLabel(record.Name)
          }
        }.bind(this));
        const sorted_unique_labels = new Map([...unique_labels.entries()].slice().sort((a, b) => b[1] - a[1]));
        // If Elasticsearch returned undefined labels then delete them:
        sorted_unique_labels.delete(undefined);
        this.countLabels(sorted_unique_labels.size, es_data.length);
        return sorted_unique_labels
      }
    },
    watch: {
      // These watches update the line chart
      selectedLabel: function() {
        this.chartData();
      },
      elasticsearch_data: function() {
        this.chartData();
      },
    },
    deactivated: function () {
      console.log('deactivated component:', this.operator);
      this.boxes_available = [];
      this.selectedLabel = '';
      clearInterval(this.canvasRefreshInterval);
      const canvas = document.getElementById('canvas');
      if (canvas) {
        const ctx = canvas.getContext('2d');
        if (ctx) ctx.clearRect(0, 0, canvas.width, canvas.height);
      }
    },
    activated: function () {
      console.log('activated component:', this.operator);
      this.fetchAssetData();
    },
    mounted: function() {
      this.getCurlCommand();
    },
    beforeUnmount: function () {
      this.high_confidence_data = [];
      this.elasticsearch_data = [];
      this.count_distinct_labels = 0;
      this.count_labels = 0;
      clearInterval(this.canvasRefreshInterval);
    },
    methods: {
      getCurlCommand() {
        this.searchQuery = 'AssetId:'+this.$route.params.asset_id+' Confidence:>'+this.Confidence+' Operator:'+this.operator;
        // get curl command to search elasticsearch
        this.curlCommand = 'awscurl -X GET --profile default --service es --region ' + this.AWS_REGION + ' \'' + this.SEARCH_ENDPOINT + '/_search?q=' + encodeURIComponent(this.searchQuery) + '\''
      },
      saveBoxedLabel(label_name){
        if (!this.boxes_available.includes(label_name)) {
          this.boxes_available.push(label_name);
        }
      },
      countLabels(unique_count, total_count) {
        this.count_distinct_labels = unique_count;
        this.count_labels = total_count;
      },
      saveFile() {
        const elasticsearch_data = this.elasticsearch_data;
        const blob = new Blob([JSON.stringify(elasticsearch_data)], {type: 'text/plain'});
        const a = document.createElement('a');
        a.download = "data.json";
        a.href = window.URL.createObjectURL(blob);
        a.dataset.downloadurl = ['text/json', a.download, a.href].join(':');
        const e = new MouseEvent('click', { view: window });
        a.dispatchEvent(e);
      },
      updateConfidence (event) {
        this.isBusy = !this.isBusy;
        this.Confidence = event.target.value;
        // TODO: move image processing to a separate component
        if (this.mediaType === "video") {
          // redraw markers on video timeline
          this.player.markers.removeAll();
        }
        this.fetchAssetData()
      },
      // updateMarkers updates markers in the video player and is called when someone clicks on a label button
      updateMarkers (label) {
        // clear canvas for redrawing
        this.boxes_available = [];
        clearInterval(this.canvasRefreshInterval);
        const canvas = document.getElementById('canvas');
        const ctx = canvas.getContext('2d');
        ctx.clearRect(0, 0, canvas.width, canvas.height);
        ctx.strokeStyle = "red";
        ctx.font = "15px Arial";
        ctx.textAlign = "center";
        ctx.textBaseline = "middle";
        ctx.fillStyle = "red";
        if (this.selectedLabel === label) {
          // keep the canvas clear canvas if user clicked the label button a second consecutive time
          this.selectedLabel = "";
          return
        }
        this.selectedLabel = label;        // initialize lists of boxes and markers to be drawn
        let boxMap = new Map();
        let markers = [];
        let es_data = this.elasticsearch_data;
        let i=0;
        es_data.forEach( function(record) {
          if (record.Name === label) {
            // Save label name overlaying on video timeline
            markers.push({'time': record.Timestamp/1000, 'text': record.Name, 'overlayText': record.Name});
            // Save bounding box info if it exists
            if (record.BoundingBox) {
              // TODO: move image processing to a separate component
              if (this.mediaType === "image") {
                const boxinfo = {
                  'instance': i,
                  'name': record.Name,
                  'confidence': (record.Confidence * 1).toFixed(2),
                  'x': record.BoundingBox.Left * canvas.width,
                  'y': record.BoundingBox.Top * canvas.height,
                  'width': record.BoundingBox.Width * canvas.width,
                  'height': record.BoundingBox.Height * canvas.height
                };
                boxMap.set(i++, [boxinfo])
              } else {
                // Use time resolution of 0.1 second
                const timestamp = Math.round(record.Timestamp / 100);
                const boxinfo = {
                  'instance': 0,
                  'timestamp': Math.ceil(record.Timestamp / 100),
                  'name': record.Name,
                  'confidence': (record.Confidence * 1).toFixed(2),
                  'x': record.BoundingBox.Left * canvas.width,
                  'y': record.BoundingBox.Top * canvas.height,
                  'width': record.BoundingBox.Width * canvas.width,
                  'height': record.BoundingBox.Height * canvas.height
                };
                boxMap.set(timestamp, [boxinfo])
              }
            }
          }
        }.bind(this));
        if (boxMap.size > 0) {
          this.drawBoxes(boxMap);
        }
        // TODO: move image processing to a separate component
        if (this.mediaType === "video") {
          // redraw markers on video timeline
          this.player.markers.removeAll();
          this.player.markers.add(markers);
        }
      },
      async fetchAssetData () {
        let query = 'AssetId:'+this.$route.params.asset_id+' Confidence:>'+this.Confidence+' Operator:'+this.operator;
        let apiName = 'search';
        let path = '/_search';
        let apiParams = {
          headers: {'Content-Type': 'application/json'},
          queryStringParameters: {'q': query, 'default_operator': 'AND', 'size': 10000}
        };
        let response = await this.$Amplify.API.get(apiName, path, apiParams);
        if (!response) {
          this.showElasticSearchAlert = true
        }
        else {
          let es_data = [];
          let result = await response;
          let data = result.hits.hits;
          if (data.length === 0 && this.Confidence > 55) {
            this.lowerConfidence = true;
            this.lowerConfidenceMessage = 'Try lowering confidence threshold'
          }
          else {
            this.lowerConfidence = false;
            for (let i = 0, len = data.length; i < len; i++) {
              es_data.push(data[i]._source)
            }
          }
          this.elasticsearch_data = JSON.parse(JSON.stringify(es_data));
          this.isBusy = false
        }
      },
      drawBoxes: function(boxMap) {
        const canvas = document.getElementById('canvas');
        const ctx = canvas.getContext('2d');
        // TODO: move image processing to a separate component
        if (this.mediaType === "image") {
          ctx.clearRect(0, 0, canvas.width, canvas.height);
          ctx.beginPath();
          ctx.strokeStyle = "red";
          ctx.font = "15px Arial";
          ctx.textAlign = "center";
          ctx.textBaseline = "middle";
          ctx.fillStyle = "red";
          // For each box instance...
          boxMap.forEach( i => {
            let drawMe = i[0];
            if (drawMe) {
              ctx.rect(drawMe.x, drawMe.y, drawMe.width, drawMe.height);
              // Draw object name and confidence score
              ctx.fillText(drawMe.name + " (" + drawMe.confidence + "%)", (drawMe.x + drawMe.width / 2), drawMe.y - 10);
              ctx.stroke();
            }
          });
          // now return so we avoid rendering any of the video related components below
          return
        }
        // If user just clicked a new label...
        if (this.canvasRefreshInterval !== undefined) {
          // ...then reset the old canvas refresh interval.
          clearInterval(this.canvasRefreshInterval)
        }
        // Look for and draw bounding boxes every 100ms
        const interval_ms = 100;
        const erase_on_iteration = 2;
        let i = 0;
        this.canvasRefreshInterval = setInterval(function () {
          i++;
          // erase old bounding boxes
          if (!this.player.paused() && i % erase_on_iteration === 0) {
            i=0;
            ctx.clearRect(0, 0, canvas.width, canvas.height);
            ctx.beginPath();
            ctx.strokeStyle = "red";
            ctx.font = "15px Arial";
            ctx.textAlign = "center";
            ctx.textBaseline = "middle";
            ctx.fillStyle = "red";
          }
          // Get current player timestamp to the nearest 1/10th second
          let player_timestamp = Math.round(this.player.currentTime()*10.0);
          // If we have a box for the player's timestamp...
          if (boxMap.has(player_timestamp)) {
            i=0;
            ctx.clearRect(0, 0, canvas.width, canvas.height);
            ctx.beginPath();
            ctx.strokeStyle = "red";
            ctx.font = "15px Arial";
            ctx.textAlign = "center";
            ctx.textBaseline = "middle";
            ctx.fillStyle = "red";
            // ...then get a list of box instances
            let instance_list = (boxMap.get(player_timestamp)).map( item => item.instance).filter((v, i, a) => a.indexOf(v) === i);
            // For each box instance...
            instance_list.forEach( i => {
              // ...get all of the boxes belonging to this instance
              // at the current timestamp.
              let boxes = boxMap.get(player_timestamp).filter(box => box.instance === i);
              boxes.forEach (drawMe => {
                if (drawMe) {
                  ctx.rect(drawMe.x, drawMe.y, drawMe.width, drawMe.height);
                  // Draw object name and confidence score
                  ctx.fillText(drawMe.name + " (" + drawMe.confidence + "%)", (drawMe.x + drawMe.width / 2), drawMe.y - 10);
                }
              })
            });
            ctx.stroke();
          }
        }.bind(this), interval_ms);
      },
      chartData() {
        let timeseries = new Map();
        function saveTimestamp (millisecond) {
          if (timeseries.has(millisecond)) {
            timeseries.set(millisecond, {"x": millisecond, "y": timeseries.get(millisecond).y + 1})
          } else {
            timeseries.set(millisecond, {"x": millisecond, "y":1})
          }
        }
        let es_data = this.elasticsearch_data;
        es_data.forEach( function(record) {
          // Define timestamp with millisecond resolution
          const millisecond = Math.round(record.Timestamp);
          if (this.selectedLabel) {
            // If label is defined, then enumerate timestamps for that label
            if (record.Name === this.selectedLabel) {
              saveTimestamp(millisecond);
            }
          } else {
            // No label has been selected, so enumerate timestamps for all label names.
            saveTimestamp(millisecond);
          }
        }.bind(this));
        //sort the timeseries map by its millisecond key
        const ordered_timeseries = new Map([...timeseries.entries()].slice().sort((a, b) => a[0] - b[0]));
        const chartTuples = Array.from(ordered_timeseries.values());
        this.$store.commit('updateTimeseries', chartTuples);
        this.$store.commit('updateSelectedLabel', this.selectedLabel);
      },
    }
  }
</script>
