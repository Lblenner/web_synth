<script lang="ts">
  import { onMount } from "svelte";
  import * as Tone from "tone";

  let synths: Tone.Synth[] = [];
  let managedTouches = new Map();
  let w: undefined | number, h: undefined | number;
  let started = false;
  let load = false;

  onMount(() => {
    return () => {
      synths.forEach((s) => s.dispose());
    };
  });

  const init = async () => {
    if (!started) {
      load = true;
      for (let i = 0; i < 5; i++) {
        synths.push(new Tone.Synth().toDestination());
      }
      await Tone.start();
      started = true;
      load = false;
    }
  };

  const new_index = () => {
    let indexes = [];
    for (let [_, managedTouch] of managedTouches) {
      indexes.push(managedTouch.index);
    }
    for (let i = 0; i < synths.length; i++) {
      if (!indexes.includes(i)) {
        return i;
      }
    }
    synths.push(new Tone.Synth().toDestination());
    return synths.length - 1;
  };

  const onTouchMove = async (e: TouchEvent) => {
    if (!started) {
      await init();
      return;
    }

    for (let touch of e.changedTouches) {
      let start = false;
      let touchIndex;

      let managedTouch = managedTouches.get(touch.identifier);

      if (managedTouch) {
        touchIndex = managedTouch.index;
      } else {
        touchIndex = new_index();
        start = true;
      }

      let x = touch.clientX;
      let y = touch.clientY;

      let [f, type] = updateSynth(x, y, touchIndex, start);

      managedTouches.set(touch.identifier, {
        ...touch,
        index: touchIndex,
        f: (f as number).toPrecision(4),
        type: type,
      });
    }

    managedTouches = managedTouches;
  };

  const onTouchEnd = (e: TouchEvent) => {
    if (!started) {
      return;
    }
    if (e.touches.length === 0) {
      for (let synth of synths) {
        synth.triggerRelease();
        managedTouches = new Map();
        managedTouches = managedTouches;
      }
      return;
    }

    for (let touch of e.changedTouches) {
      let managedTouch = managedTouches.get(touch.identifier);
      if (managedTouch) {
        synths[managedTouch.index].triggerRelease();
        managedTouches.delete(touch.identifier);
      }
    }
    managedTouches = managedTouches;
  };

  const onMouseUp = (e: MouseEvent) => {
    if (!started) {
      return;
    }
    synths[0].triggerRelease();
    managedTouches = new Map();
    managedTouches = managedTouches;
  };

  const onMouse = async (e: MouseEvent, start: boolean) => {
    if (e.buttons != 1) {
      return;
    }

    if (!started) {
      await init();
      return;
    }

    let x = e.clientX;
    let y = e.clientY;

    let [f, type] = updateSynth(x, y, 0, start);

    managedTouches.set(0, {
      ...e,
      index: 0,
      f: (f as number).toPrecision(4),
      type: type,
    });
    managedTouches = managedTouches;
  };

  const updateSynth = (
    x: number,
    y: number,
    synthIndex: number,
    start: boolean
  ) => {
    if (h && w) {
      let f = (y / h) * 530 + 70;
      let ratio = x / w;
      let type;

      for (let i = 1; i < 33; i++) {
        if (ratio < i / 32) {
          type = "sawtooth" + i.toString();
          synths[synthIndex].oscillator.set({
            type: type as "sawtooth",
          });
          break;
        }
      }

      if (start) synths[synthIndex].triggerAttack(f);
      else synths[synthIndex].setNote(f);

      return [f, type];
    }
    return [0, 0];
  };
</script>

<div>
  <div class="absolute w-full h-screen gradient1"></div>
  <div class="absolute w-full h-screen gradient2"></div>
  <div
    class="absolute w-full h-screen pointer-events-none flex justify-center items-center flex-col"
  >
    {#if load}
      chargement...
    {/if}
    {#if !started && !load}
      <div class="bg-orange-400 p-2 rounded-md text-center">
        cliquer pour commencer
      </div>
    {/if}
    <div class="grid grid-cols-3 gap-4 p-2">
      {#each managedTouches as [_, touch]}
        <div class="bg-orange-400 flex items-center flex-col p-2 rounded-md">
          <div>{touch.index}</div>
          <div>{touch.f}</div>
          <div class="px-1">{touch.type}</div>
        </div>
      {/each}
    </div>
  </div>
  <canvas
    bind:clientWidth={w}
    bind:clientHeight={h}
    on:touchmove={(e) => onTouchMove(e)}
    on:touchstart={(e) => onTouchMove(e)}
    on:touchend={onTouchEnd}
    on:touchcancel={onTouchEnd}
    on:mousedown={(e) => onMouse(e, true)}
    on:mousemove={(e) => onMouse(e, false)}
    on:mouseup={onMouseUp}
    class="w-full h-screen touch-none transparent z-10"
  >
  </canvas>
</div>
