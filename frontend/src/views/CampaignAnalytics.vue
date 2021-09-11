<template>
  <section class="analytics content relative">
    <h1 class="title is-4">{{ $t('analytics.title') }}</h1>
    <hr />

    <form @submit.prevent="onSubmit">
      <div class="columns">
        <div class="column is-6">
          <b-field :label="$t('globals.terms.campaigns')" label-position="on-border">
            <b-taginput v-model="form.campaigns" :data="queriedCampaigns" name="campaigns" ellipsis
              icon="tag-outline" :placeholder="$t('globals.terms.campaigns')"
              autocomplete :allow-new="false" :before-adding="isCampaignSelected"
              @typing="queryCampaigns" field="name" :loading="isSearchLoading"></b-taginput>
          </b-field>
        </div>

        <div class="column is-5">
          <div class="columns">
            <div class="column is-6">
              <b-field data-cy="from" :label="$t('analytics.fromDate')" label-position="on-border">
                <b-datetimepicker
                  v-model="form.from"
                  icon="calendar-clock"
                  :timepicker="{ hourFormat: '24' }"
                  :datetime-formatter="formatDateTime" @input="onFromDateChange" />
              </b-field>
            </div>
            <div class="column is-6">
              <b-field data-cy="to" :label="$t('analytics.toDate')" label-position="on-border">
                <b-datetimepicker
                  v-model="form.to"
                  icon="calendar-clock"
                  :timepicker="{ hourFormat: '24' }"
                  :datetime-formatter="formatDateTime" @input="onToDateChange" />
              </b-field>
            </div>
          </div><!-- columns -->
        </div><!-- columns -->

        <div class="column is-1">
          <b-button native-type="submit" type="is-primary" icon-left="magnify"
            :disabled="form.campaigns.length === 0" data-cy="btn-search"></b-button>
        </div>
      </div><!-- columns -->
    </form>

    <section class="charts mt-6">
      <div class="chart" v-for="(v, k) in charts" :key="k">
        <b-loading v-if="v.loading" :active="v.loading" :is-full-page="false" />
        <h4 v-if="v.chart !== null">{{ v.name }}</h4>
        <div :ref="`chart-${k}`" :id="`chart-${k}`"></div>
      </div>
    </section>
  </section>
</template>

<style lang="css">
  @import "~c3/c3.css";
</style>

<script>
import Vue from 'vue';
import dayjs from 'dayjs';
import c3 from 'c3';

const colors = [
  '#734FC3',
  '#FFB50D',
  '#41AC9C',
  '#ee7d5b',
  '#7FC7BC',
  '#3a82d6',
  '#688ED9',
  '#FFC43D',
];

export default Vue.extend({
  data() {
    return {
      isSearchLoading: false,
      queriedCampaigns: [],

      // data for each view.
      charts: {
        views: {
          name: this.$t('campaigns.views'),
          data: [],
          chart: null,
          loading: false,
          fn: this.$api.getCampaignViewCounts,
          dataFn: this.processLines,
        },

        clicks: {
          name: this.$t('campaigns.clicks'),
          data: [],
          chart: null,
          loading: false,
          fn: this.$api.getCampaignClickCounts,
          dataFn: this.processLines,
        },

        bounces: {
          name: this.$t('globals.terms.bounces'),
          data: [],
          chart: null,
          loading: false,
          fn: this.$api.getCampaignBounceCounts,
          dataFn: this.processLines,
        },

        links: {
          name: this.$t('analytics.links'),
          data: [],
          chart: null,
          loading: false,
          fn: this.$api.getCampaignLinkCounts,
          dataFn: this.processLinks,
        },
      },

      form: {
        campaigns: [],
        from: null,
        to: null,
      },
    };
  },

  methods: {
    formatDateTime(s) {
      return dayjs(s).format('YYYY-MM-DD HH:mm');
    },

    isCampaignSelected(camp) {
      return !this.form.campaigns.find(({ id }) => id === camp.id);
    },

    onFromDateChange() {
      if (this.form.from > this.form.to) {
        this.form.to = dayjs(this.form.from).add(7, 'day').toDate();
      }
    },

    onToDateChange() {
      if (this.form.from > this.form.to) {
        this.form.from = dayjs(this.form.to).add(-7, 'day').toDate();
      }
    },

    renderLineChart(typ, data, el) {
      const conf = {
        bindto: el,
        unload: true,
        data: {
          type: 'spline',
          xs: {},
          columns: [],
          names: [],
          colors: {},
          empty: { label: { text: this.$t('globals.messages.emptyState') } },
        },
        axis: {
          x: {
            type: 'timeseries',
            tick: {
              format: '%Y-%m-%d %H:%M',
            },
          },
        },
        legend: {
          show: false,
        },
      };

      // Add campaign data to the chart.
      data.forEach((c, n) => {
        if (c.data.length === 0) {
          return;
        }

        const x = `x${n + 1}`;
        const d = `data${n + 1}`;

        // data1, data2, datan => x1, x2, xn.
        conf.data.xs[d] = x;

        // Campaign name for each datan.
        conf.data.names[d] = c.name;

        // Dates for each xn.
        conf.data.columns.push([x, ...c.data.map((v) => dayjs(v.timestamp))]);

        // Counts for each datan.
        conf.data.columns.push([d, ...c.data.map((v) => v.count)]);

        // Colours for each datan.
        conf.data.colors[d] = colors[n % data.length];
      });

      this.$nextTick(() => {
        if (this.charts[typ].chart) {
          this.charts[typ].chart.destroy();
        }

        this.charts[typ].chart = c3.generate(conf);
      });
    },

    processLinks(typ, camps, data) {
      const conf = {
        bindto: this.$refs[`chart-${typ}`][0],
        unload: true,
        data: {
          type: 'bar',
          x: 'x',
          columns: [],
          color: (c, d) => (typeof (d) === 'object' ? colors[d.index % data.length] : colors[0]),
          empty: { label: { text: this.$t('globals.messages.emptyState') } },
          onclick: (d) => {
            window.open(data[d.index].url, '_blank', 'noopener noreferrer');
          },
        },
        bar: {
          width: {
            max: 30,
          },
        },
        axis: {
          rotated: true,
          x: {
            type: 'category',
            tick: {
              multiline: false,
            },
          },
        },
      };

      // Add link data to the chart.
      // https://c3js.org/samples/axes_x_tick_rotate.html
      conf.data.columns.push(['x', ...data.map((l) => {
        try {
          const u = new URL(l.url);
          if (l.url.length > 80) {
            return `${u.hostname}${u.pathname.substr(0, 50)}..`;
          }
          return u.hostname + u.pathname;
        } catch {
          return l.url;
        }
      })]);
      conf.data.columns.push([this.$t('analytics.count'), ...data.map((l) => l.count)]);

      this.$nextTick(() => {
        if (this.charts[typ].chart) {
          this.charts[typ].chart.destroy();
        }
        this.charts[typ].chart = c3.generate(conf);
      });
    },

    processLines(typ, camps, data) {
      // Make a campaign id => camp lookup map to group incoming
      // data by campaigns.
      const campIDs = camps.reduce((obj, c) => {
        const out = { ...obj };
        out[c.id] = c;
        return out;
      }, {});

      // Group individual data points per campaign id.
      // {1: [...], 2: [...]}
      const groups = data.reduce((obj, d) => {
        const out = { ...obj };
        if (!(d.campaignId in out)) {
          out[d.campaignId] = [];
        }

        out[d.campaignId].push(d);
        return out;
      }, {});

      Object.keys(groups).forEach((k) => {
        this.charts[typ].data.push({
          name: campIDs[groups[k][0].campaignId].name,
          data: groups[k],
        });
      });

      this.$nextTick(() => {
        this.renderLineChart(typ, this.charts[typ].data, this.$refs[`chart-${typ}`][0]);
      });
    },

    onSubmit() {
      // Fetch count for each analytics type (views, counts, bounces);
      Object.keys(this.charts).forEach((k) => {
        // Clear existing data.
        this.charts[k].data = [];

        // Fetch views, clicks, bounces for every campaign.
        this.getData(k, this.form.campaigns);
      });
    },

    queryCampaigns(q) {
      this.isSearchLoading = true;
      this.$api.getCampaigns({
        query: q,
        order_by: 'created_at',
        order: 'DESC',
      }).then((data) => {
        this.isSearchLoading = false;
        this.queriedCampaigns = data.results.map((c) => {
          // Change the name to include the ID in the auto-suggest results.
          const camp = c;
          camp.name = `#${c.id}: ${c.name}`;
          return camp;
        });
      });
    },

    getData(typ, camps) {
      this.charts[typ].loading = true;

      // Call the HTTP API.
      this.charts[typ].fn({
        id: camps.map((c) => c.id),
        from: this.form.from,
        to: this.form.to,
      }).then((data) => {
        this.charts[typ].dataFn(typ, camps, data);
        this.charts[typ].loading = false;
      });
    },
  },

  created() {
    const now = dayjs().set('hour', 23).set('minute', 59).set('seconds', 0);
    this.form.to = now.toDate();
    this.form.from = now.subtract(7, 'day').set('hour', 0).set('minute', 0).toDate();
  },
});
</script>
