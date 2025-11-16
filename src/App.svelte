<script lang="ts">
  import logo from "/dndlog.png";

  type Mode = "dm" | "player";
  type Time = { hour: number, minute: number, second: number };
  type DateTime = { day: number } & Time;
  type Preset = { label?: string, amount: DateTime };
  type Event = { when: DateTime, summary: string, dmOnly: boolean, description: string|null };

  type StateVersion1 = {
      version: 1,
      mode: Mode,
      now: DateTime,
      events: Event[],
      presets: Preset[],
  }

  const defaultState = {
      version: 1,
      mode: "dm",
      now: { day: 1, hour: 12, minute: 0, second: 0 },
      events: [],
      presets: [
          {
              label: "Combat Round",
              amount: { day: 0, hour: 0, minute: 0, second: 6}
          },
          {
              label: "Short Rest",
              amount: {day: 0, hour: 1, minute: 0, second: 0},
          },
          {
              label: "Long Rest",
              amount: {day: 0, hour: 8, minute: 0, second: 0},
          }
      ]
  } as StateVersion1;

  let s = $state(loadState());

  let moment = $derived.by(() => formatDateTime(s.now));

  function formatAdvance(by : DateTime) {
      let components = [];
      if (by.day > 0) {
          components.push(`${by.day} day${by.day === 1 ? '' : 's'}`);
      }
      if (by.hour > 0) {
          components.push(`${by.hour} hour${by.hour === 1 ? '' : 's'}`);
      }
      if (by.minute > 0) {
          components.push(`${by.minute} minute${by.minute === 1 ? '' : 's'}`);
      }
      if (by.second > 0) {
          components.push(`${by.second} second${by.second === 1 ? '' : 's'}`);
      }

      if (components.length === 0) {
          return "None";
      }
      if (components.length === 1) {
          return components[0];
      }
      if (components.length === 2) {
          return `${components[0]} and ${components[1]}`;
      }
      return components.slice(0, components.length-1).join(", ") + ", and " + components.slice(components.length-1);
  }

  function formatDateTime(dateTime : DateTime) {
      return `Day ${dateTime.day} - ${formatTime(dateTime)}`;
  }
  function formatTime({ hour, minute, second }: Time) {
      return `${hour < 10 ? "0" : ""}${hour}:${minute < 10 ? "0" : ""}${minute}:${second < 10 ? "0" : ""}${second}`;
  }

  function advanceTime(from : DateTime, by : DateTime) {
      let newDate = {
          day: from.day + by.day,
          hour: from.hour + by.hour,
          minute: from.minute + by.minute,
          second: from.second + by.second,
      }
      if (newDate.second >= 60) {
          newDate.minute += Math.floor(newDate.second / 60);
          newDate.second %= 60;
      }
      if (newDate.minute >= 60) {
          newDate.hour += Math.floor(newDate.minute / 60);
          newDate.minute %= 60;
      }
      if (newDate.hour >= 24) {
          newDate.day += Math.floor(newDate.hour / 24);
          newDate.hour %= 24;
      }

      return newDate;
  }
  function advanceNow(by : DateTime) {
      s.now = advanceTime(s.now, by);
  }

  function addPreset(this : HTMLFormElement, event : SubmitEvent) {
      event.preventDefault();

      let label = (this.elements.namedItem("label")! as HTMLInputElement).value;

      s.presets.push({
          label: label ? label : undefined,
          amount: {
              day: (this.elements.namedItem("days")! as HTMLInputElement).valueAsNumber,
              hour: (this.elements.namedItem("hours")! as HTMLInputElement).valueAsNumber,
              minute: (this.elements.namedItem("minutes")! as HTMLInputElement).valueAsNumber,
              second: (this.elements.namedItem("seconds")! as HTMLInputElement).valueAsNumber,
          },
      });

      this.reset();
  }

  function addEntry(this : HTMLFormElement, event : SubmitEvent) {
      event.preventDefault();

      const description = (this.elements.namedItem("description")! as HTMLInputElement).value;

      s.events.unshift({
          when: s.now,
          summary: (this.elements.namedItem("summary")! as HTMLTextAreaElement).value,
          dmOnly: (this.elements.namedItem("dm-only")! as HTMLInputElement).checked,
          description: description ? description : null,
      })

      this.reset();
  }

  function storeState(state : StateVersion1) {
      if (isDefaultState(state)) {
          return;
      }

      history.pushState(null, '', getStateUrl(state));
  }

  function getStateUrl(state : StateVersion1) {
      let url = new URL(window.location.toString());
      url.searchParams.set('state', encodeState(state));
      return url;
  }

  function loadState() : StateVersion1 {
    const params = new URLSearchParams(window.location.search);
    const state = params.get('state');
    if (state === null) {
        return defaultState;
    }

    return decodeState(state);
  }

  function encodeState(state : StateVersion1) : string {
      return btoa(JSON.stringify(state));
  }


  function decodeState(encoded : string) : StateVersion1 {
      try {
          let state = JSON.parse(atob(encoded));
          // Additional change of mode field.
          if (typeof state.mode === "undefined") {
              state.mode = "dm";
          }
          return state;
      }
      catch (e) {
          if (e instanceof DOMException) {
              console.error("Error decoding base64 string", e);
          }
          else if (e instanceof SyntaxError) {
              console.error("Error decoding JSON string", e);
          }
          else {
              console.error("Unexpected error", e);
          }
          return defaultState;
      }
  }

  function isDefaultState(state : StateVersion1) {
      if (state.now.day !== defaultState.now.day) {
          return false;
      }
      if (state.now.hour !== defaultState.now.hour) {
          return false;
      }
      if (state.now.minute !== defaultState.now.minute) {
          return false;
      }
      if (state.now.second !== defaultState.now.second) {
          return false;
      }
      if (state.events.length !== defaultState.events.length) {
          return false;
      }
      if (state.presets.length !== defaultState.presets.length) {
          return false;
      }
      return true;
  }

  $effect(() => storeState(s))

  function copyDMUrl() {
    navigator.clipboard.writeText(getStateUrl(s).toString());
  }

  function copyPlayerUrl() {
      navigator.clipboard.writeText(getStateUrl(filterPlayerState(s)).toString());
  }

  function filterPlayerState(state: StateVersion1) : StateVersion1 {
      return {
          version: 1,
          mode: "player",
          now: state.now,
          // DM Presets may contain secrets.
          presets: defaultState.presets,
          events: state.events.filter(e => !e.dmOnly),
      }
  }

</script>

<main class={s.mode === "dm" ? "dm-view" : "player-view"}>
  <div id="header">
    <img src={logo} width="56" alt="DND Log"/>

    <h1>
      {#if s.mode === "player"}
        Events up to
      {/if}
      {moment}
    </h1>

    {#if s.mode === "dm"}
      <div id="shortcuts">
        <button type="button" onclick={copyDMUrl}>Copy DM URL</button>
        <button type="button" onclick={copyPlayerUrl}>Copy Player URL</button>
      </div>
    {/if}
  </div>

  {#if s.mode === "dm"}
    <div id="advance-time">
      <h2>Advance Time</h2>
      <div class="presets">
        {#each s.presets as preset (preset.label ?? formatAdvance(preset.amount))}
          <button type="button" onclick={() => advanceNow(preset.amount)}>
            {#if preset.label}
              {preset.label} ({formatAdvance(preset.amount)})
            {:else}
              {formatAdvance(preset.amount)}
            {/if}
          </button>
        {/each}
      </div>

      <details id="add-preset">
        <summary>Add preset</summary>
        <form onsubmit={addPreset}>
            <label>
              Label
              <input type="text" name="label" />
            </label>
            <div class="selector">
              <label>
                Days
                <input required value="0" min="0" type="number" name="days" />
              </label>
              <label>
                Hours
                <input required value="0" min="0" max="23" type="number" name="hours" />
              </label>
              <label>
                Minutes
                <input required value="0" min="0" max="59" type="number" name="minutes" />
              </label>
              <label>
                Seconds
                <input required value="0" min="0" max="59" type="number" name="seconds" />
              </label>
            </div>

            <input type="submit" value="Add preset" />
        </form>
      </details>
    </div>

    <div id="add-entry">
    <h2>Add entry</h2>
    <form onsubmit={addEntry}>
      <label>
        Summary
        <input type="text" name="summary" required />
      </label>
      <label>
        <input type="checkbox" name="dm-only" value="1" checked />
        DM Only
      </label>
      <label class="stacked">
        Description
        <textarea name="description"></textarea>
      </label>

      <input type="submit" value="Add">
    </form>
    </div>
  {/if}

  <div id="timeline">
    <h2>Timeline</h2>
    <ul>
    {#each s.events as event (event.when, event.summary)}
      <li>
      {#if event.description}
        <details>
          <summary>{formatDateTime(event.when)} {event.dmOnly ? "(DM Only)" : ""} - {event.summary}</summary>
          {event.description}
        </details>
      {:else}
        <span>{formatDateTime(event.when)} {event.dmOnly ? "(DM Only)" : ""} - {event.summary}</span>
      {/if}
      </li>
    {:else}
      <em>Nothing happened yet, add your first entry to add it to the current time.</em>
    {/each}
    </ul>

  </div>
</main>

<style>
  h1, h2 {
    margin: 0;
  }

  summary {
    cursor: pointer;
  }

  main {
    padding: 1rem;
  }

  @media (min-width: 800px) {
    main.dm-view {
      display: grid;
      grid-template-columns: repeat(9, 1fr);
      grid-auto-rows: minmax(100px, auto);
      grid-template-areas:
        "hdr  hdr  hdr  hdr   hdr   hdr   hdr   hdr   hdr  "
        "adv  adv  adv  entry entry entry entry entry entry"
        "adv  adv  adv  log   log   log   log   log   log  "
        "adv  adv  adv  log   log   log   log   log   log  "
      ;
      gap: 1rem;

      #header {
        grid-area: hdr;
        align-content: center;
      }
    }
  }

  .stacked {
    display: flex;
    flex-direction: column;
  }

  #header {
    display: flex;
    flex-wrap: wrap;
    gap: 1rem;
    align-items: center;
    justify-content: center;

    @media (min-width: 600px) {
      img {
        width: 76px;
      }
    }

    @media (min-width: 900px) {
      justify-content: space-between;

      img {
        width: 108px;
      }
      h1 {
        margin-right: auto;
      }
    }
  }

  #shortcuts {
    display: flex;
    gap: 1rem;
    justify-content: flex-end;
    align-items: center;
  }

  #advance-time {
    grid-area: adv;

    .presets {
      display: flex;
      flex-direction: column;
      gap: .5rem;
    }

    .presets + details {
      margin-top: 1rem;
    }
  }

  #add-preset {
    padding: 1rem;

    &:open {
      border: 1px solid black;
    }

    .selector {
      display: flex;
      flex-direction: column;
      gap: .5rem;
      justify-content: center;

      label {
        display: flex;
        flex-direction: column;
      }

      input {
        width: 5rem;
      }
    }
  }

  #add-entry {
    grid-area: entry;

    form {
      display: flex;
      flex-direction: column;
    }
  }

  #timeline {
    grid-area: log;

    summary::marker {
      content: "";
    }
  }
</style>
