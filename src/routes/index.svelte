<script lang="ts">
  import type { CountdownDate } from "$lib/types";
  import { getDates, dates } from "../stores/dates";
  import Timer from "./_timer.svelte";
  import { onMount, onDestroy } from "svelte";
  import { addSeconds, compareAsc, format, formatISO, parseISO } from "date-fns";
  import { browser } from "$app/env";

  import { fade } from "svelte/transition";

  let now : Date;
  let activeDate : CountdownDate;
  $: dateMarker = activeDate == null ? null : parseISO(activeDate.date);
  $: if (now > dateMarker && !fetching) {
    updateDates();
  }
  let backoff = 5;
  let fetching = false;

  function clamp(value : number, min : number, max : number) {
    return Math.min(Math.max(value, min), max);
  }

  let previousUpdate : NodeJS.Timeout;

  async function updateDates() {
    if (previousUpdate != null) {
      clearTimeout(previousUpdate);
    }
    fetching = true;
    if (activeDate?.id === -1) {
      activeDate = null;
      dateMarker = null;
    }
    let hadError = false;
    try {
      $dates = await getDates();
      $dates = $dates.filter(
        (d) => compareAsc(now, parseISO(d.date)) < 0
      );
    } catch (error) {
      hadError = true;
      console.error(error);
    }

    setActiveDate(hadError);
    previousUpdate = setTimeout(updateDates, 60 * 1000);
    fetching = false;
  }

  function getAdditionalBackoffAmount(number : Number) {
    if (number < 15) {
      return 5;
    }
    if (number < 30) {
      return 15;
    }
    if (number < 60) {
      return 30;
    }
    return 60;
  }

  function setActiveDate(hadError : Boolean) {
    if ($dates.length === 0) {
      activeDate = {
        id: -1,
        date: formatISO(addSeconds(now, backoff + 1)),
        title: hadError ? "Error getting countdown data, retrying in..." : "No countdown found, refreshing in...",
        description: "",
      };
      backoff = clamp(backoff + getAdditionalBackoffAmount(backoff), 5, 60);
    } else {
      backoff = 5;
      activeDate = $dates[0];
    }
  }

  let animationRequest : number;
  function updateTime(timestamp : DOMHighResTimeStamp) {
    now = new Date();
    animationRequest = requestAnimationFrame(updateTime);
  }

  let visible = browser ? false : true;
  onMount(() => {
    if (browser) now = new Date();
    updateDates();
    if (browser) animationRequest = window.requestAnimationFrame(updateTime);
    visible = true;
  });

  onDestroy(() => {
    clearTimeout(previousUpdate);
    if (browser) window.cancelAnimationFrame(animationRequest);
  })
</script>

<div class="flex flex-col justify-center items-center h-[100vh] max-h-[100vh] w-[100vw]">
  <div class="flex-grow flex flex-col justify-center items-center py-8 w-[100vw] dark:text-zinc-50">
    {#if visible}
      <div>
        <p class="text-center text-3xl md:text-4xl lg:text-5xl">
          <b>Current Countdown:</b>
        </p>
        {#key activeDate?.id}
          <p class="mt-1 text-center text-xl md:text-2xl lg:text-3xl text-ow2-orange dark:text-ow2-light-orange" in:fade="{{duration: 500, delay: 200}}">
            {activeDate ? activeDate.title : "Loading..."}
          </p>
          <p class="text-center text-md md:text-xl lg:text-2xl" in:fade="{{duration: 500, delay: 500}}">
            {activeDate && activeDate.id != -1 ? format(parseISO(activeDate?.date), "PPPPp") : " "}
          </p>
        {/key}
      </div>
      <Timer start={now} end={dateMarker} />
    {/if}
  </div>
  <div class="flex justify-between my-4 mx-2 px-4 h-8 w-full">
    <p class="text-xs text-zinc-500 dark:text-zinc-400 italic text-left">Created by <a class="text-ow2-orange dark:text-ow2-light-orange underline" href="https://twitter.com/Cactus_Puppy" rel="noreferrer noopener" target="_blank">@Cactus_Puppy</a><br /><a href="https://github.com/CactusPuppy/ow2countdown" rel="noreferrer noopener" target="_blank" class="text-ow2-orange dark:text-ow2-light-orange underline">GitHub</a></p>
    <p class="text-xs text-zinc-500 dark:text-zinc-400 italic text-right">
      This site and its creator are not affiliated with Overwatch or Blizzard Entertainment.
      <br />Overwatch 2 and the Overwatch 2 logo are ©2022 Blizzard Entertainment, Inc.
    </p>
  </div>
</div>
