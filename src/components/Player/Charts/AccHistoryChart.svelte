<script>
  import Chart from 'chart.js/auto'
  import {formatNumber} from '../../../utils/format'
  import {
    formatDate,
    formatDateWithOptions,
    toSsMidnight,
  } from '../../../utils/date'
  import createPlayerService from '../../../services/scoresaber/player'
  import {debounce} from '../../../utils/debounce'
  import {onLegendClick} from './utils/legend-click-handler'
  import {getContext} from 'svelte'
  import {DateTime} from 'luxon'

  export let playerId = null;
  export let rankHistory = null;
  export let height = "350px";

  const CHART_DAYS = 50;
  const CHART_DEBOUNCE = 300;

  const pageContainer = getContext('pageContainer');

  const playerService = createPlayerService();

  let canvas = null;
  let chart = null;

  let lastHistoryHash = null;
  let playerHistory = null;

  const calcHistoryHash = (accHistory) =>
    (accHistory ? Object.values(accHistory).map(h => Object.values(h).join(',')).join(':') : '')
  ;

  async function refreshPlayerHistory(playerId) {
    if (!playerId) return;

    playerHistory = await playerService.getPlayerHistory(playerId) ?? null;
  }

  async function setupChart(hash, canvas) {
    if (!hash || !canvas || chartHash === lastHistoryHash) return;

    lastHistoryHash = chartHash;

    if (rankHistory.length < CHART_DAYS) rankHistory = Array(CHART_DAYS - rankHistory.length).fill(null).concat(rankHistory);

    const gridColor = '#2a2a2a'
    const averageColor = '#3273dc';
    const medianColor = '#8992e8';
    const stdDevColor = '#f94022';
    const ssPlusColor = 'rgba(143,72,219, .4)';
    const ssColor = 'rgba(190,42,66, .4)';
    const sPlusColor = 'rgba(255,99,71, .4)';
    const sColor = 'rgba(89,176,244, .4)';
    const aColor = 'rgba(60,179,113, .4)';

    const datasets = [];

    const dtAccSaberToday = DateTime.fromJSDate(toSsMidnight(new Date()));
    const dayTimestamps = Array(CHART_DAYS).fill(0).map((_, idx) => toSsMidnight(dtAccSaberToday.minus({days: CHART_DAYS - 1 - idx}).toJSDate()).getTime());

    const xAxis = {
      type: 'time',
      display: true,
      offset: true,
      time: {
        unit: 'day',
      },
      scaleLabel: {
        display: false,
      },
      ticks: {
        autoSkip: false,
        major: {
          enabled: true,
        },
        font: function (context) {
          if (context.tick && context.tick.major) {
            return {
              weight: 'bold',
            };
          }
        },
        callback: (val, idx, ticks) => {
          if (!ticks?.[idx]) return '';

          return formatDateWithOptions(new Date(ticks[idx]?.value), {
            localeMatcher: 'best fit',
            day: '2-digit',
            month: 'short',
          });
        },
      },
      grid: {
        color: gridColor,
      },
    };

    const isScoreDataAvailable = playerHistory && playerHistory.find(h => !!h.avgAcc);

    let yAxes = {
      y: {
        display: true,
        position: 'left',
        title: {
          display: $pageContainer.name !== 'phone',
          text: 'Acc',
        },
        ticks: {
          callback: val => formatNumber(val, 2) + '%',
          precision: 2
        },
        grid: {
          color: gridColor,
        },
      },
    };

    if (additionalHistory && Object.keys(additionalHistory).length) {
      if (isScoreDataAvailable)
        yAxes = {
          ...yAxes,
          y1: {
            display: true,
            position: 'right',
            title: {
              display: $pageContainer.name !== 'phone',
              text: 'Maps count',
            },
            ticks: {
              callback: val => val === Math.floor(val) ? val : null,
              precision: 0,
            },
            grid: {
              drawOnChartArea: false,
            },
          },

          y2: {
            display: false,
            position: 'right',
            title: {
              display: $pageContainer.name !== 'phone',
              text: 'Std dev',
            },
            ticks: {
              callback: val => val,
              precision: 3,
            },
            grid: {
              color: gridColor,
            },
          },
        }

      const skipped = (ctx, value) => ctx.p0.skip || ctx.p1.skip ? value : undefined;

      [
        {
          key: 'avgAcc',
          secKey: 'averageRankedAccuracy',
          name: 'Average',
          borderColor: averageColor,
          round: 2,
          axis: 'y',
          axisOrder: 0,
        },
      ]
        .concat(
          isScoreDataAvailable
            ? [
              {key: 'medianAcc', name: 'Median', borderColor: medianColor, round: 2, axis: 'y', axisOrder: 1},
              {key: 'stdDeviation', name: 'Std dev', borderColor: stdDevColor, round: 4, axis: 'y2', axisOrder: 10},
              {
                key: 'accBadges', keys: [
                  {key: 'SS+', name: 'SS+', backgroundColor: ssPlusColor, axisOrder: 2},
                  {key: 'SS', name: 'SS', backgroundColor: ssColor, axisOrder: 3},
                  {key: 'S+', name: 'S+', backgroundColor: sPlusColor, axisOrder: 4},
                  {key: 'S', name: 'S', backgroundColor: sColor, axisOrder: 5},
                  {key: 'A', name: 'A', backgroundColor: aColor, axisOrder: 5},
                ], round: 0, axis: 'y1',
              },
            ]
            : [],
        )
        .forEach(obj => {
          const {key, secKey, keys, name, axis, ...options} = obj;
          if (keys && Array.isArray(keys)) {
            keys.forEach(obj => {
              const {key: innerKey, name, ...innerOptions} = obj;
              const fieldData = dayTimestamps.map(x => ({x, y: additionalHistory?.[x]?.[key]?.[innerKey] ?? 0}));

              datasets.push({
                ...options,
                ...innerOptions,
                yAxisID: axis,
                label: name,
                data: fieldData,
                fill: true,
                borderWidth: 2,
                pointRadius: 1,
                cubicInterpolationMode: 'monotone',
                tension: 0.4,
                type: 'bar',
                stack: key,
                spanGaps: true,
                segment: {
                  borderWidth: ctx => skipped(ctx, 1),
                  borderDash: ctx => skipped(ctx, [6, 6]),
                },
              });
            })
          } else {
            const fieldData = dayTimestamps.map(x => ({x, y: additionalHistory?.[x]?.[key] ?? null}));

            datasets.push({
              ...options,
              yAxisID: axis,
              label: name,
              data: fieldData,
              fill: false,
              borderWidth: 2,
              pointRadius: 1,
              cubicInterpolationMode: 'monotone',
              tension: 0.4,
              type: 'line',
              spanGaps: true,
              segment: {
                borderWidth: ctx => skipped(ctx, 1),
                borderDash: ctx => skipped(ctx, [6, 6]),
              },
            });
          }
        });
    }

    if (!chart)
    {
      chart = new Chart(
            canvas,
            {
              type: 'line',
              data: {datasets},
              options: {
                responsive: true,
                maintainAspectRatio: false,
                layout: {
                  padding: {
                    right: 0
                  }
                },
                interaction: {
                  mode: 'index',
                  intersect: false,
                },
                plugins: {
                  legend: {
                    display: true,
                    onClick: onLegendClick,
                  },
                  tooltip: {
                    position: 'nearest',
                    callbacks: {
                      title(ctx) {
                        if (!ctx?.[0]?.raw) return '';

                        const nextDayDate = DateTime.fromMillis(ctx[0].raw?.x).plus({days: 1}).toJSDate();
                        const nextDayDateFormatted = nextDayDate > new Date() ? 'now' : formatDate(nextDayDate, 'short', 'short');

                        return `${formatDate(new Date(ctx[0].raw?.x), 'short', 'short')} - ${nextDayDateFormatted}`;
                      },
                      label(ctx) {
                        switch (ctx.dataset.label) {
                          case 'SS+':
                          case 'SS':
                          case 'S+':
                          case 'S':
                          case 'A':
                            return ` ${ctx.dataset.label}: ${formatNumber(ctx.parsed.y, ctx.dataset.round)}`

                          default:
                            return ` ${ctx.dataset.label}: ${formatNumber(ctx.parsed.y, ctx.dataset.round)}%`;
                        }
                      },
                    }
                  },
                },
                scales: {
                  x: xAxis,
                  ...yAxes,
                },
              },
            },
          );
    }
    else {
      chart.data = {datasets};
      chart.options.scales = {x: xAxis, ...yAxes};
      chart.update();
    }
  }

  let debouncedChartHash = null;
  const debounceChartHash = debounce(chartHash => debouncedChartHash = chartHash, CHART_DEBOUNCE);

  $: refreshPlayerHistory(playerId);

  $: additionalHistory = playerHistory && playerHistory.length
    ? playerHistory.reduce((cum, h) => {
      const time = toSsMidnight(h.ssDate)?.getTime()
      if (!time) return cum;

      const history = {[time]: {averageRankedAccuracy: h. averageRankedAccuracy, avgAcc: h.avgAcc, medianAcc: h.medianAcc, stdDeviation:h.stdDeviation, accBadges: h.accBadges}};

      return {...cum, ...history};
    }, {})
    : null;

  $: chartHash = calcHistoryHash(rankHistory, additionalHistory);
  $: debounceChartHash(chartHash)
  $: if (debouncedChartHash) setupChart(debouncedChartHash, canvas)
</script>

{#if rankHistory && rankHistory.length}
  <section class="chart" style="--height: {height}">
    <canvas class="chartjs" bind:this={canvas} height={parseInt(height,10)}></canvas>
  </section>
{/if}

<style>
    section {
        position: relative;
        margin: 1rem auto 0 auto;
        height: var(--height, 300px);
    }

    canvas {
        width: 100% !important;
    }
</style>
