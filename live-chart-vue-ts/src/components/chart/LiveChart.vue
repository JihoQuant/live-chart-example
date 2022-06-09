<script setup lang="ts">
import { ref, onMounted, onUnmounted } from "vue";
import {
  createChart,
  CrosshairMode,
  CandlestickData,
  IChartApi,
  ISeriesApi,
  UTCTimestamp,
} from "lightweight-charts";

interface Tick extends CandlestickData {
  code: string;
  time: UTCTimestamp;
}

const chartConatiner = ref<HTMLElement | null>(null);

// eslint-disable-next-line @typescript-eslint/no-explicit-any
const data = ref<Array<Tick>>([]);
const lastTick = ref<Tick>();

const socket = ref<WebSocket>();
const response = ref(false);

const requestBody = {
  header: {
    appkey: "PSoBI9B37V1feYvchx7XBrE1H7tSJle2hpS0",
    appsecret:
      "zgO37wescpeTjuuaKcnKBBtfURWHvw4pl783AXUDRb728adzruOG9q4WJjPN6f73ZxZkCjZHzW2sjNmPsH1wb5ayro4FxeUriLLnhb8oasHE2YYQdQ6933ki1rEO6VpV5RYUGEnWdGOBFfA9JfdC/zPH4+8iFr78KxWSFZ3GwbkksO2aA8Q=",
    custtype: "P",
    tr_type: "1",
    "content-type": "utf-8",
  },
  body: {
    input: {
      tr_id: "H0STCNT0",
      tr_key: "005930",
    },
  },
};

onMounted(() => {
  if (chartConatiner.value == null) return;

  let chart: IChartApi;
  let series: ISeriesApi<"Candlestick">;

  // socket.value = new WebSocket("ws://localhost:3000");
  socket.value = new WebSocket("ws://ops.koreainvestment.com:21000");

  socket.value.addEventListener("open", () => {
    socket.value?.send(JSON.stringify(requestBody));

    if (chartConatiner.value) {
      chart = createChart(chartConatiner.value, {
        width: window.innerWidth,
        height: 300,
        crosshair: {
          mode: CrosshairMode.Normal,
        },
      });

      series = chart.addCandlestickSeries();
      series.setData(data.value);
    }
  });

  socket.value.addEventListener("message", (e) => {
    console.log({ value: e.data });

    try {
      const payload = JSON.parse(e.data);
      if (!response.value) {
        response.value = payload;
      }
      return;
    } catch {
      const body = (e.data as string).split("|")[3];
      const values = body.split("^");
      const close = parseInt(values[2]);
      const time = parseInt(values[1]) as UTCTimestamp;

      data.value.pop();

      if (lastTick.value) {
        if (
          Math.floor(time / 100) != Math.floor(lastTick.value.time / 100) &&
          lastTick.value.open != 0
        ) {
          data.value.push({ ...lastTick.value, close });
          lastTick.value.open = close;
          lastTick.value.high = close;
          lastTick.value.low = close;
        }

        if (lastTick.value.high < close) {
          lastTick.value.high = close;
        }

        if (lastTick.value.low > close) {
          lastTick.value.low = close;
        }

        lastTick.value = {
          ...lastTick.value,
          code: values[0],
          time: time,
          close,
        };
      } else {
        lastTick.value = {
          code: values[0],
          open: close,
          high: close,
          low: close,
          close,
          time,
        };
      }

      data.value.push({ ...lastTick.value });

      series.setData(data.value);
    }
  });
});

onUnmounted(() => {
  socket.value?.close();
});
</script>

<template>
  <div ref="chartConatiner"></div>
  <div>
    {{ response }}
    <p v-for="item in data" :key="item.time">
      {{ item }}
    </p>
  </div>
</template>
