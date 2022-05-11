<script setup lang="ts">
import { computed, ref, onMounted, unref, useAttrs } from "vue";
const attrs = useAttrs();
const color = ref(attrs.color);
const el = ref<HTMLCanvasElement>();
const ctx = computed(() => el!.value?.getContext("2d")!);
const WIDTH = 600;
const HEIGHT = 600;
const LENGTH = 20;
onMounted(() => {
  init();
});

interface Point {
  x: number;
  y: number;
}

interface Branch {
  start: Point;
  length: number;
  theta: number;
}

function lineTo(p1: Point, p2: Point) {
  unref(ctx).lineWidth = Math.abs(Math.random() * 1) + 0.5;
  unref(ctx).lineCap = "round";
  unref(ctx).beginPath();
  unref(ctx).moveTo(p1.x, p1.y);
  unref(ctx).lineTo(p2.x, p2.y);
  unref(ctx).stroke();
}

function drawBranch(b: Branch) {
  lineTo(b.start, getEndPoint(b));
}

function getEndPoint(b: Branch) {
  return {
    x: b.start.x + b.length * Math.cos(b.theta),
    y: b.start.y + b.length * Math.sin(b.theta),
  };
}

const pendingTasks: Function[] = [];

function init() {
  unref(ctx).strokeStyle = color.value;
  step({
    start: { x: WIDTH / 4, y: HEIGHT },
    length: LENGTH,
    theta: -Math.PI / 2,
  });
  step({
    start: { x: (3 * WIDTH) / 4, y: 0 },
    length: LENGTH,
    theta: Math.PI / 2,
  });
}
function boundary(p: Point) {
  if (p.x < 0 || p.x > WIDTH || p.y > HEIGHT || p.y < 0) {
    return false;
  }
  return true;
}

function step(b: Branch, depth = 0) {
  const end = getEndPoint(b);
  drawBranch(b);
  if ((depth < 4 || Math.random() < 0.5) && boundary(end)) {
    pendingTasks.push(() =>
      step(
        {
          start: end,
          length: b.length + Math.random() * 10 - 5,
          theta: b.theta - 0.4 * Math.random(),
        },
        depth + 1
      )
    );
  }
  if ((depth < 4 || Math.random() < 0.5) && boundary(end)) {
    pendingTasks.push(() =>
      step(
        {
          start: end,
          length: b.length + Math.random() * 10 - 5,
          theta: b.theta + 0.4 * Math.random(),
        },
        depth + 1
      )
    );
  }
}

function frame() {
  const tasks = [...pendingTasks];
  pendingTasks.length = 0;
  tasks.forEach((fn) => fn());
}
let framesCount = 0;
function startFrame() {
  requestAnimationFrame(() => {
    framesCount += 1;
    if (framesCount % 2 === 0) {
      frame();
    }
    startFrame();
  });
}
startFrame();
</script>

<template>
  <canvas
    ref="el"
    width="600"
    height="600"
    :style="{ border: `1px solid ${color}` }"
  ></canvas>
</template>

<style>
body {
  margin: 0;
  padding: 0;
  text-align: center;
}
#app {
  width: 100%;
  height: 100vh;
}
</style>
